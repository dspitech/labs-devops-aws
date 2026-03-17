# LAB : EBS Volumes, Snapshots, and AMI Management

## Contexte

Ce document explique comment déployer une infrastructure simple sur AWS en utilisant **AWS CLI**.

L'objectif de ce TP est de :

- créer une **clé SSH**
- créer un **Security Group**
- lancer une **instance EC2**
- créer un **volume EBS supplémentaire** (Volume)
- attacher ce volume à l'instance
- créer un **snapshot** (Instantanés)
- créer une **AMI**

Ces étapes permettent de comprendre la gestion du **stockage et des ressources EC2** via la ligne de commande.

---

## Architecture

```
EC2 Instance (t2.micro)
│
├── Volume principal
│
└── Volume EBS supplémentaire (8 Go)
        │
        └── Snapshot de sauvegarde
```

## Remarque

Ce lab doit être réalisé dans **AWS CloudShell**.

## 1. Création de la clé SSH

Une clé SSH est nécessaire pour se connecter à une instance EC2.

```bash
# Création de clé
aws ec2 create-key-pair --key-name DevOpsKey --query 'KeyMaterial' --output text > DevOpsKey.pem

# Gestion des autorisations
chmod 400 DevOpsKey.pem
```

![image](https://hackmd.io/_uploads/r1ESgNlcbg.png)

![image](https://hackmd.io/_uploads/S1jDgEgc-x.png)

### Explication

- **create-key-pair** : crée une paire de clés AWS
- **--key-name DevOpsKey** : nom de la clé
- **--query 'KeyMaterial'** : récupère uniquement la clé privée
- **--output text** : sortie au format texte
- **> DevOpsKey.pem** : enregistre la clé dans un fichier

## 2. Création du Security Group

Un **Security Group** agit comme un pare-feu pour l'instance **EC2**.

Avant de créer le groupe, il faut récupérer l'ID du **VPC** :

```bash
aws ec2 describe-vpcs
```

![image](https://hackmd.io/_uploads/ByF2e4g5-x.png)

![image](https://hackmd.io/_uploads/S1Pk-4l9Wl.png)

### Création du Security Group

```Bash
aws ec2 create-security-group \
  --group-name DevOpsSG \
  --description "SG TP Stockage" \
  --vpc-id vpc-099163b43e6f5403d
```

Il faut remplacer vpc-id par votre propre ID

![image](https://hackmd.io/_uploads/HyDSZVgcZx.png)

![image](https://hackmd.io/_uploads/r1PwZNgc-l.png)

### Explication

- **--group-name** : nom du groupe de sécurité
- **--description** : description du groupe
- **--vpc-id** : identifiant du VPC dans lequel le groupe sera créé (à remplacer par votre propre vpc-id)

## 3. Autorisation du port SSH

On récupère d'abord l'ID du Security Group.

```Bash
SG_ID=$(aws ec2 describe-security-groups \
--filters "Name=group-name,Values=DevOpsSG" \
--query "SecurityGroups[0].GroupId" \
--output text)
```

Ensuite on autorise l'accès SSH.

```Bash
aws ec2 authorize-security-group-ingress \
--group-id $SG_ID \
--protocol tcp \
--port 22 \
--cidr 0.0.0.0/0
```

![image](https://hackmd.io/_uploads/ryPAbElqWl.png)

![image](https://hackmd.io/_uploads/BJCkfElqbl.png)

### Explication

- **--protocol tcp** : protocole réseau
- **--port 22** : port SSH
- **--cidr 0.0.0.0/0** : accès depuis n'importe quelle adresse IP

⚠️ **Attention :** en production, il est recommandé de limiter l'accès à votre adresse IP.

## 4. Création de l'instance EC2

On lance une instance dans la région Paris (eu-west-3).

```Bash
aws ec2 run-instances \
--image-id ami-05d43d5e94bb6eb95 \
--count 1 \
--instance-type t2.micro \
--key-name DevOpsKey \
--security-group-ids $SG_ID \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=InstanceStockageTP}]' \
--placement "AvailabilityZone=eu-west-3a"

```

![image](https://hackmd.io/_uploads/rJwOMEeqZl.png)

![image](https://hackmd.io/_uploads/ByVcz4eqbg.png)

### Explication

- **--image-id** : AMI utilisée pour créer l'instance
- **--count 1** : nombre d'instances
- **--instance-type t2.micro** : type d'instance
- **--key-name** : clé SSH utilisée pour se connecter
- **--security-group-ids** : groupe de sécurité associé
- **--tag-specifications** : ajoute un tag pour identifier l'instance
- **--placement** : zone de disponibilité

## 5. Création d'un volume EBS

On crée un volume supplémentaire de 8 Go.

```Bash
aws ec2 create-volume \
--size 8 \
--volume-type gp3 \
--availability-zone eu-west-3a \
--tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=VolumeExtra}]'
```

![image](https://hackmd.io/_uploads/HJ5pzVl9Zx.png)

![image](https://hackmd.io/_uploads/rJQb74xcZe.png)

### Explication

- **--size 8** : taille du volume en Go
- **--volume-type gp3** : volume SSD performant
- **--availability-zone** : doit être identique à celle de l'instance.

## 6. Récupération automatique des IDs

On récupère automatiquement :

- l'ID de l'instance
- l'ID du volume

```Bash
INSTANCE_ID=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=InstanceStockageTP" \
"Name=instance-state-name,Values=running,pending" \
--query "Reservations[0].Instances[0].InstanceId" \
--output text)

VOL_ID=$(aws ec2 describe-volumes \
--filters "Name=tag:Name,Values=VolumeExtra" \
--query "Volumes[0].VolumeId" \
--output text)
```

![image](https://hackmd.io/_uploads/HkKX7Vx9-l.png)

## 7. Attachement du volume à l'instance

```Bash
aws ec2 attach-volume \
--volume-id $VOL_ID \
--instance-id $INSTANCE_ID \
--device /dev/sdf
```

![image](https://hackmd.io/_uploads/B14SQElcWx.png)

![image](https://hackmd.io/_uploads/rJ_wXVgqWg.png)

### Explication

- **--volume-id** : identifiant du volume
- **--instance-id** : identifiant de l'instance
- **--device** : point d'attachement

Sur Linux, le volume apparaîtra généralement sous : `/dev/xvdf`

## 8. Création d'un snapshot

Un snapshot est une sauvegarde du volume EBS.

```Bash
aws ec2 create-snapshot \
--volume-id $VOL_ID \
--description "Backup de mon volume a Paris"
```

![image](https://hackmd.io/_uploads/r1OpmEg5-x.png)

Voir l'état du snapshot

```
aws ec2 describe-snapshots --snapshot-ids snap-06442ba7b2dba20e7 --query "Snapshots[0].State"
```

Remplacer par votre propre id snap

![image](https://hackmd.io/_uploads/BJKLVNxcbl.png)

![image](https://hackmd.io/_uploads/SkynVVgqZg.png)

### Avantages

- **Sauvegarde incrémentale**
- **Restauration rapide**
- **Possibilité de recréer un volume à partir du snapshot**

## 9. Création d'une AMI

Une AMI (Amazon Machine Image) permet de créer des copies identiques d'une instance.

```Bash
aws ec2 create-image \
--instance-id $INSTANCE_ID \
--name "DevOpsServerAMI-Paris" \
--no-reboot
```

![image](https://hackmd.io/_uploads/rkmA4Nl5bx.png)

![image](https://hackmd.io/_uploads/rJw-r4g5be.png)

### Explication

- **create-image** : crée une image de l'instance
- **--name** : nom de l'image
- **--no-reboot** : évite de redémarrer l'instance

## 10. Vérification des volumes attachés

```Bash
aws ec2 describe-volumes \
--filters "Name=attachment.instance-id,Values=$INSTANCE_ID" \
--query "Volumes[*].{ID:VolumeId,Etat:State,Device:Attachments[0].Device}" \
--output table
```

![image](https://hackmd.io/_uploads/ryI4r4gcZg.png)

### Cette commande affiche

- **l'ID du volume**
- **son état**
- **le périphérique attaché**
- Etat `in-use`, ça signifie que le volume est actuellement attaché à une instance EC2 et utilisé activement.

## 11 : Nettoyage

Pour éviter les coûts, il faut supprimer **toutes les ressources créées** : instances, volumes, snapshots, AMI, security groups et clés SSH.

---

### 1. Terminer l'instance EC2

Récupération de l'ID de l'instance (si pas déjà défini) :

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=InstanceStockageTP" \
"Name=instance-state-name,Values=running,pending,stopped" \
--query "Reservations[0].Instances[0].InstanceId" \
--output text)
```

![image](https://hackmd.io/_uploads/BkH9SVe5Wx.png)

![image](https://hackmd.io/_uploads/S1F3rNxqZl.png)

Terminer l'instance :

```Bash
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

⚠️ Vérifier que l'instance est bien terminée avant de supprimer les volumes attachés.

![image](https://hackmd.io/_uploads/Bklk8ExqZe.png)

![image](https://hackmd.io/_uploads/ryxZ84e9Wx.png)

![image](https://hackmd.io/_uploads/BJbELVgqbe.png)

### 2. Supprimer les volumes EBS

Récupération des volumes associés :

```Bash
VOL_IDS=$(aws ec2 describe-volumes \
--filters "Name=tag:Name,Values=VolumeExtra" \
--query "Volumes[*].VolumeId" \
--output text)
```

![image](https://hackmd.io/_uploads/r1TwUNlqZx.png)

Supprimer chaque volume

```Bash
for VOL_ID in $VOL_IDS; do
    aws ec2 delete-volume --volume-id $VOL_ID
done
```

![image](https://hackmd.io/_uploads/BkE5UNecbx.png)

![image](https://hackmd.io/_uploads/HyCo8Nxqbx.png)

### 3. Supprimer l’AMI et ses snapshots associés

Récupération de l’AMI :

```Bash
AMI_ID=$(aws ec2 describe-images \
--owners self \
--filters "Name=name,Values=DevOpsServerAMI-Paris" \
--query "Images[0].ImageId" \
--output text)
```

![image](https://hackmd.io/_uploads/SkxeYElcbx.png)

Déréférencer et supprimer l’AMI :

```Bash
aws ec2 deregister-image --image-id $AMI_ID
```

![image](https://hackmd.io/_uploads/BkH2i4lcZl.png)

![image](https://hackmd.io/_uploads/HkWRj4l5bx.png)

### 4. Supprimer les snapshots

Liste tous les snapshots correspondant à ta description :

```Bash
aws ec2 describe-snapshots --filters "Name=description,Values='Backup de mon volume a Paris'" --query "Snapshots[*].[SnapshotId,State]" --output table
```

![image](https://hackmd.io/_uploads/BkJeOVlc-e.png)

Supprimer chaque snapshot :

```Bash
SNAP_IDS=$(aws ec2 describe-snapshots --filters "Name=description,Values='Backup de mon volume a Paris'" --query "Snapshots[*].SnapshotId" --output text)

for SNAP_ID in $SNAP_IDS; do
    echo "Suppression du snapshot $SNAP_ID..."
    aws ec2 delete-snapshot --snapshot-id $SNAP_ID
    # Attente que le snapshot disparaisse
    while aws ec2 describe-snapshots --snapshot-ids $SNAP_ID --query "Snapshots[*].SnapshotId" --output text | grep -q $SNAP_ID; do
        echo "En attente de la suppression de $SNAP_ID..."
        sleep 3
    done
    echo "Snapshot $SNAP_ID supprimé."
done
```

![image](https://hackmd.io/_uploads/B1eZnVl9-e.png)
![image](https://hackmd.io/_uploads/rycu3El9bl.png)

### 5. Supprimer le Security Group

Récupération de l’ID du SG :

```Bash
SG_ID=$(aws ec2 describe-security-groups \
--filters "Name=group-name,Values=DevOpsSG" \
--query "SecurityGroups[0].GroupId" \
--output text)
```

![image](https://hackmd.io/_uploads/BkqjhExqZx.png)

Suppression :

```Bash
aws ec2 delete-security-group --group-id $SG_ID
```

![image](https://hackmd.io/_uploads/Sk763Ne5bl.png)

### 6. Supprimer la clé SSH

```Bash
aws ec2 delete-key-pair --key-name DevOpsKey
rm -f DevOpsKey.pem
```

![image](https://hackmd.io/_uploads/BJTGT4xqZe.png)

### 7. Vérification finale

Lister toutes les ressources restantes :

```Bash
# Instances
INSTANCES=$(aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,State.Name,Tags]" --output json)
if [ "$INSTANCES" != "[]" ]; then
    echo "$INSTANCES" | jq
else
    echo "⚠️ Aucune instance trouvée"
fi

# Volumes
VOLUMES=$(aws ec2 describe-volumes --query "Volumes[*].[VolumeId,State,Tags]" --output json)
if [ "$VOLUMES" != "[]" ]; then
    echo "$VOLUMES" | jq
else
    echo "⚠️ Aucun volume trouvé"
fi

# Snapshots
SNAPS=$(aws ec2 describe-snapshots --owner-ids self --query "Snapshots[*].[SnapshotId,Description]" --output json)
if [ "$SNAPS" != "[]" ]; then
    echo "$SNAPS" | jq
else
    echo "⚠️ Aucun snapshot trouvé"
fi

# Security Groups
SGS=$(aws ec2 describe-security-groups --query "SecurityGroups[*].[GroupId,GroupName]" --output json)
if [ "$SGS" != "[]" ]; then
    echo "$SGS" | jq
else
    echo "⚠️ Aucun Security Group trouvé"
fi

# Clés
KEYS=$(aws ec2 describe-key-pairs --query "KeyPairs[*].KeyName" --output json)
if [ "$KEYS" != "[]" ]; then
    echo "$KEYS" | jq
else
    echo "⚠️ Aucune clé trouvée"
fi
```

![image](https://hackmd.io/_uploads/SJSJRNg9bg.png)

✅ Résultat

Après ces commandes, toutes les ressources créées pour le TP sont supprimées, ce qui permet de ne plus générer de frais AWS.

## Bonus

Voici un script Bash complet pour un nettoyage automatique de toutes les ressources créées dans ce TP AWS EC2. Tu pourras le copier dans un fichier `cleanup.sh` et l’exécuter une seule fois.

```Bash
#!/bin/bash
set -e
echo "✅ Début du nettoyage sécurisé..."

# Variables
INSTANCE_TAG="InstanceStockageTP"
VOLUME_TAG="VolumeExtra"
SNAPSHOT_DESC="Backup de mon volume a Paris"
AMI_NAME="DevOpsServerAMI-Paris"
SECURITY_GROUP="DevOpsSG"
KEY_NAME="DevOpsKey"

# --- 1. Terminer l'instance ---
INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=$INSTANCE_TAG" "Name=instance-state-name,Values=running,pending,stopped" \
    --query "Reservations[].Instances[].InstanceId" --output text)

if [ -n "$INSTANCE_ID" ]; then
    echo "🔹 Terminaison de l'instance $INSTANCE_ID..."
    aws ec2 terminate-instances --instance-ids $INSTANCE_ID
    aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
    echo "✅ Instance terminée."
else
    echo "⚠️ Aucune instance trouvée."
fi

# --- 2. Supprimer l’AMI ---
AMI_ID=$(aws ec2 describe-images --owners self --filters "Name=name,Values=$AMI_NAME" --query "Images[].ImageId" --output text)
if [ -n "$AMI_ID" ]; then
    echo "🔹 Désenregistrement de l'AMI $AMI_ID..."
    aws ec2 deregister-image --image-id $AMI_ID
    echo "✅ AMI supprimée."
else
    echo "⚠️ Aucune AMI trouvée."
fi

# --- 3. Supprimer les snapshots ---
SNAP_IDS=$(aws ec2 describe-snapshots --filters "Name=description,Values=$SNAPSHOT_DESC" --query "Snapshots[].SnapshotId" --output text)
for SNAP_ID in $SNAP_IDS; do
    echo "🔹 Suppression du snapshot $SNAP_ID..."
    aws ec2 delete-snapshot --snapshot-id $SNAP_ID || echo "⚠️ Impossible de supprimer $SNAP_ID"
done
echo "✅ Snapshots supprimés."

# --- 4. Supprimer les volumes ---
VOL_IDS=$(aws ec2 describe-volumes --filters "Name=tag:Name,Values=$VOLUME_TAG" --query "Volumes[].VolumeId" --output text)
for VOL_ID in $VOL_IDS; do
    echo "🔹 Suppression du volume $VOL_ID..."
    aws ec2 delete-volume --volume-id $VOL_ID
    aws ec2 wait volume-deleted --volume-ids $VOL_ID
done
echo "✅ Volumes supprimés."

# --- 5. Supprimer le Security Group ---
SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=$SECURITY_GROUP" --query "SecurityGroups[].GroupId" --output text)
if [ -n "$SG_ID" ]; then
    echo "🔹 Suppression du Security Group $SG_ID..."
    aws ec2 delete-security-group --group-id $SG_ID
    echo "✅ Security Group supprimé."
else
    echo "⚠️ Aucun Security Group trouvé."
fi

# --- 6. Supprimer la clé SSH ---
aws ec2 delete-key-pair --key-name $KEY_NAME || echo "⚠️ Clé $KEY_NAME inexistante"
rm -f $KEY_NAME.pem
echo "✅ Clé SSH supprimée."

# --- 7. Vérification finale ---
echo "🔹 Vérification des ressources restantes..."
aws ec2 describe-instances --query "Reservations[].Instances[].InstanceId" --output table || echo "Aucune instance"
aws ec2 describe-volumes --query "Volumes[].VolumeId" --output table || echo "Aucun volume"
aws ec2 describe-snapshots --owner-ids self --query "Snapshots[].SnapshotId" --output table || echo "Aucun snapshot"
aws ec2 describe-security-groups --query "SecurityGroups[].GroupId" --output table || echo "Aucun Security Group"
aws ec2 describe-key-pairs --query "KeyPairs[].KeyName" --output table || echo "Aucune clé"

echo "✅ Nettoyage complet terminé !"
```

### Instructions d’utilisation

- Copier ce script dans un fichier `cleanup.sh`.
- Rendre le script exécutable :

```Bash
chmod +x cleanup.sh
```

- Exécuter le script :

```Bash
./cleanup.sh
```

### Le script va automatiquement

- **Terminer l’instance EC2**
- **Supprimer tous les volumes EBS associés**
- **Supprimer les snapshots**
- **Supprimer l’AMI**
- **Supprimer le Security Group**
- **Supprimer la clé SSH locale et AWS**
- **Vérifier l’absence de ressources restantes**
