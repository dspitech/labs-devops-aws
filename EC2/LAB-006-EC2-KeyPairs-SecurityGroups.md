# LAB : EC2 Instances, Key Pairs, and Security Groups

Ce TP a pour objectif de vous apprendre à provisionner une instance Linux sécurisée en utilisant uniquement la ligne de commande.

## Objectifs

À la fin de ce TP, tu seras capable de :

- Créer une **paire de clés SSH** pour accéder à l’instance
- Restreindre les permissions de la clé pour la sécurité (`chmod 400`)
- Identifier ton **VPC par défaut**
- Créer un **groupe de sécurité** et autoriser le trafic SSH (port 22)
- Lancer une instance EC2 avec AMI Amazon Linux, Key Pair et Security Group
- Récupérer et afficher les informations de l’instance (ID, VPC, subnet, security group, AMI, tags, etc.)
- Te connecter à l’instance via SSH
- Supprimer l’instance, la Key Pair et le Security Group après usage
- Vérifier que tout nettoyage a été correctement effectué

---

## Prérequis

Avant de commencer, tu dois avoir :

- Un **compte AWS actif**
- **AWS CLI** installé et configuré (`aws configure`)
- **SSH** installé pour se connecter à l’instance
- **Permissions EC2** pour créer instances, Key Pairs et Security Groups
- Terminal Bash ou CloudShell AWS

## Remarque

Ce lab doit être réalisé dans **AWS CloudShell**.

---

## Étape 1 : Création des accès (Key Pair)

```Bash
aws ec2 create-key-pair --key-name DevOpsKey --query 'KeyMaterial' --output text > DevOpsKey.pem
```

![image](https://hackmd.io/_uploads/BkFsszgcWl.png)

![image](https://hackmd.io/_uploads/HkcTjfeqWe.png)

## Etape 2 : Restreindre les permissions du fichier (Sécurité)

```Bash
chmod 400 DevOpsKey.pem
```

![image](https://hackmd.io/_uploads/BJpfhMgc-l.png)

## Etape 3 : Trouver votre VPC ID

```Bash
aws ec2 describe-vpcs --query "Vpcs[*].{ID:VpcId, Default:IsDefault}" --output table
```

![image](https://hackmd.io/_uploads/r1gShflqZl.png)

![image](https://hackmd.io/_uploads/rJtt3zx9be.png)

## Etape 4 : Création du groupe

```Bash
aws ec2 create-security-group \
--group-name DevOpsSG \
--description "SG pour TP EC2" \
--vpc-id vpc-099163b43e6f5403d
```

> Pour le vpc-id : remplacer le par votre propre ID

![image](https://hackmd.io/_uploads/BySzTGe9bg.png)

![image](https://hackmd.io/_uploads/H1YBpMlq-e.png)

## Etape 5 : Récupération de l'id du groupe

```Bash
aws ec2 describe-security-groups \
--filters "Name=group-name,Values=DevOpsSG" \
--query "SecurityGroups[*].GroupId" \
--output text
```

![image](https://hackmd.io/_uploads/HJ9OpGl5-e.png)

## Etape 6 : Réglage des ports (22 SSH) et permissions

```Bash
# On autorise le port 22 pour SSH
aws ec2 authorize-security-group-ingress --group-id sg-08217b4422a851670 --protocol tcp --port 22 --cidr 0.0.0.0/0
```

> Pour le group-id : remplacer le par votre propre ID-Group

![image](https://hackmd.io/_uploads/S1K1Cze9Wx.png)

![image](https://hackmd.io/_uploads/HkuZ0Ge9-x.png)

## Etape 7 : Récupération des infos (VPC-ID, Group-ID, Key-name, Subnet-ID)

```Bash
echo "--- RÉCAPITULATIF DE VOS RESSOURCES ---" && \
echo "VPC ID (Défaut) : " $(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text) && \
echo "Security Group ID : " $(aws ec2 describe-security-groups --filters "Name=group-name,Values=DevOpsSG" --query "SecurityGroups[0].GroupId" --output text) && \
echo "Key Pair Name : " $(aws ec2 describe-key-pairs --filters "Name=key-name,Values=DevOpsKey" --query "KeyPairs[0].KeyName" --output text) && \
echo "Subnet ID : " $(aws ec2 describe-subnets --filters "Name=default-for-az,Values=true" --query "Subnets[0].SubnetId" --output text)
```

![image](https://hackmd.io/_uploads/HkCyy7g9bl.png)

## Etape 8 : Lancement du EC2

> Rempalcer par vos vrais iidentifiants : Key-name, Group-ID, Subnet-ID

> L'image (AMI) de l'instance choisi est l'image par défaut

![image](https://hackmd.io/_uploads/HJU1eQgcZl.png)

On passe à la création de l'instance

```Bash
aws ec2 run-instances \
--image-id ami-05d43d5e94bb6eb95 \
--count 1 \
--instance-type t2.micro \
--key-name DevOpsKey \
--security-group-ids sg-08217b4422a851670 \
--subnet-id subnet-0a43e7ed25d656dad \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MonServeurDevOps}]'
```

![image](https://hackmd.io/_uploads/r1lBgQg9Zx.png)

![image](https://hackmd.io/_uploads/SymweXe5-x.png)

## Etape 9 : Se connecter à l'instance

![image](https://hackmd.io/_uploads/SkS2gXg5-g.png)

![image](https://hackmd.io/_uploads/rkkWWmgcWe.png)

## Etape 10 : Supression

### Terminer l'instance (Destruction)

```Bash
# On récupère l'ID de l'instance via son nom pour la supprimer
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=MonServeurDevOps" "Name=instance-state-name,Values=running,pending,stopped" --query "Reservations[0].Instances[0].InstanceId" --output text)

aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

![image](https://hackmd.io/_uploads/Sy41GQlcZg.png)

![image](https://hackmd.io/_uploads/HyaxG7gqbx.png)

### Supprimer le Security Group

```Bash
# 1. On récupère l'ID via le nom "DevOpsSG"
SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=DevOpsSG" --query "SecurityGroups[0].GroupId" --output text)

# 2. On le supprime
aws ec2 delete-security-group --group-id $SG_ID
```

⚠️ **Pourquoi ça pourrait échouer ?**

Si vous recevez un message d'erreur de type `DependencyViolation`, c'est pour l'une de ces deux raisons :

- **L'instance est encore "en vie"** : Elle doit être à l'état `terminated` (totalement disparue de la console) avant que le groupe soit libéré.

- **Règles croisées** : Un autre groupe de sécurité possède une règle qui fait référence à celui-ci.

![image](https://hackmd.io/_uploads/SJJ9MQl5Zl.png)

![image](https://hackmd.io/_uploads/Bk7hzQx9-l.png)

### Supprimer la Key Pair

```Bash
# Supprime la clé du catalogue AWS
aws ec2 delete-key-pair --key-name DevOpsKey

# Supprime le fichier sur votre ordinateur
rm DevOpsKey.pem
```

![image](https://hackmd.io/_uploads/rkd_QXe9bx.png)

![image](https://hackmd.io/_uploads/ry2Fm7xcWl.png)

### 🛡️ Vérification finale

```Bash
echo "--- VÉRIFICATION FINALE DU NETTOYAGE ---"
echo "1. Instances actives:"
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running,pending,stopped" --query "Reservations[*].Instances[*].InstanceId" --output table

echo "2. Groupes de sécurité 'DevOpsSG'  :"
aws ec2 describe-security-groups --filters "Name=group-name,Values=DevOpsSG" --query "SecurityGroups[*].GroupId" --output table

echo "3. Clé 'DevOpsKey'  :"
aws ec2 describe-key-pairs --filters "Name=key-name,Values=DevOpsKey" --query "KeyPairs[*].KeyName" --output table

echo "4. Fichier local .pem :"
ls DevOpsKey.pem 2>/dev/null || echo "Le fichier local a bien été supprimé."

```

![image](https://hackmd.io/_uploads/rkMoNXl9bg.png)

### Le dernier petit piège à vérifier : Les Volumes EBS

Parfois, si l'instance n'était pas configurée pour "supprimer le volume à la terminaison", le disque dur reste et coûte de l'argent. Vérifie ici :

```Bash
aws ec2 describe-volumes --query "Volumes[*].{ID:VolumeId,State:State}" --output table
```

![image](https://hackmd.io/_uploads/BkL7H7lc-x.png)
