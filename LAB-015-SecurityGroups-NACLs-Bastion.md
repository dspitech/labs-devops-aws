# LAB : Security Groups, NACLs, and Bastion Hosts

L'objectif est de créer un **Bastion** (un pont sécurisé) pour accéder à des instances qui n'ont pas d'adresse IP publique et qui sont cachées derrière un **pare-feu réseau (NACL)**.

## Concepts clés

## VPC (Virtual Private Cloud)

Un **VPC** est un réseau virtuel privé dans AWS, isolé du reste des clients.  
Il permet de contrôler les adresses IP, les subnets, les tables de routage, et les passerelles Internet.

- **CIDR Block** : définit la plage d'adresses IP privées pour le VPC, par exemple `10.0.0.0/16`.
- **Avantages** : isolation réseau, contrôle des accès, segmentation des ressources.

---

## Subnets : public et privé

Un **subnet** est une subdivision du VPC.

### Subnet Public

- Accessible depuis Internet si un **Internet Gateway (IGW)** est attaché au VPC.
- Attribue automatiquement une **IP publique** aux instances si configuré.
- Utilisé pour les **Bastion Hosts, Load Balancers**.

### Subnet Privé

- Non accessible directement depuis Internet.
- Les instances n'ont **pas d'IP publique**.
- Utilisé pour les serveurs backend ou bases de données.
- Accès à Internet possible via un **NAT Gateway**.

---

## Bastion Host

Un **Bastion Host** est une instance exposée sur Internet qui sert de **point de passage sécurisé** pour accéder aux instances privées.

- Utilisé pour **SSH vers des instances privées**.
- Placé dans le **subnet public**.
- Protégé par un **Security Group** qui autorise SSH depuis certaines IPs seulement.

---

## Security Groups (SG)

Un **Security Group** agit comme un **pare-feu au niveau de l’instance**.

- **Règles entrantes (Ingress)** : qui peut se connecter à l’instance.
- **Règles sortantes (Egress)** : vers où l’instance peut se connecter.
- **État (Stateful)** : si une connexion entrante est autorisée, la réponse sortante est automatiquement autorisée.

### Exemple :

- Bastion SG : SSH depuis l’IP publique de l’utilisateur ou `0.0.0.0/0` pour tests.
- Private SG : SSH uniquement depuis le Bastion (via son SG_ID).

---

## Clés SSH

Une **clé SSH** permet de se connecter aux instances EC2.

- AWS fournit un fichier `.pem` lors de la création d’une **Key Pair**.
- Il faut protéger le fichier : `chmod 400 DevOpsKey.pem`.
- Utilisé lors du SSH :
  ```bash
  ssh -i DevOpsKey.pem ec2-user@IP_INSTANCE
  ```

## Network ACLs (NACL)

Un **Network ACL** est un **pare-feu au niveau du subnet**.

- **Stateless** : chaque règle doit être définie pour l’aller **ET** le retour.
- Plus **granulaire** que les Security Groups pour filtrer le trafic réseau.

### Exemple de règles

- Autoriser **HTTP (port 80)** entrant
- Bloquer tout le reste **par défaut**

---

## Internet Gateway (IGW) et Routage

Une **Internet Gateway (IGW)** permet aux instances dans un **subnet public** d’accéder à Internet.

Pour qu’un **subnet public** soit accessible depuis Internet :

- Le **VPC** doit avoir une **IGW attachée**
- La **table de routage** du subnet doit contenir une **route vers l’IGW**
- Les **instances doivent avoir une IP publique**

---

## 8️⃣ Architecture

```
VPC 10.0.0.0/16
│
├─ Subnet Public 10.0.1.0/24
│   └─ Bastion Host (IP publique)
│
└─ Subnet Privé 10.0.2.0/24
    └─ Private Server (IP privée)
```

- Le **Bastion** permet d’accéder aux instances privées via **SSH**
- **Security Groups** et **NACL** contrôlent le trafic entrant et sortant
- Les **clés SSH** sécurisent les connexions
- L’**Internet Gateway** connecte le subnet public à Internet

---

## Points clés à retenir

- **Subnets publics** : instances exposées à Internet
- **Subnets privés** : instances sécurisées, non accessibles directement depuis Internet
- **Bastion** : point d’accès sécurisé aux ressources privées
- **Security Groups** : pare-feu de l’instance (**stateful**)
- **NACL** : pare-feu du subnet (**stateless**)
- **Clés SSH** : méthode sécurisée d’accès aux instances
- **IGW** : permet l’accès Internet aux subnets publics

## Remarque

Ce lab doit être réalisé dans **AWS CloudShell**.

## Déploiement

### création du VPC

La commande suivante crée un **VPC (Virtual Private Cloud)** avec une plage d’adresses IP privées et récupère automatiquement son identifiant.

```Bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
```

![image](https://hackmd.io/_uploads/HkuPoPgc-x.png)

![image](https://hackmd.io/_uploads/ry8nsvl9Ze.png)

#### Explication

- **aws ec2 create-vpc** : crée un VPC dans votre compte AWS
- **--cidr-block 10.0.0.0/16** : définit la plage d’adresses IP privées pour le VPC
- **--query 'Vpc.VpcId'** : récupère uniquement l’identifiant du VPC
- **--output text** : formate la sortie en texte brut pour pouvoir la stocker dans une variable shell
- **VPC_ID=$(…)** : stocke l’ID du VPC dans la variable `VPC_ID` pour l’utiliser dans les étapes suivantes

### Création du Subnet Public (pour le Bastion)

La commande suivante crée un **subnet public** dans le VPC précédemment créé et récupère son identifiant.

```Bash
PUB_SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone eu-west-3a --query 'Subnet.SubnetId' --output text)
```

![image](https://hackmd.io/_uploads/Byh0jPxcZl.png)

![image](https://hackmd.io/_uploads/Sy_W3Pg9-g.png)

#### Explication

- **aws ec2 create-subnet** : crée un subnet dans un VPC existant
- **--vpc-id $VPC_ID** : identifiant du VPC où sera créé le subnet
- **--cidr-block 10.0.1.0/24** : plage d’adresses IP privées du subnet
- **--availability-zone eu-west-3a** : zone de disponibilité AWS où le subnet sera situé
- **--query 'Subnet.SubnetId'** : récupère uniquement l’identifiant du subnet
- **--output text** : formate la sortie en texte brut pour l’utiliser dans une variable shell
- **PUB_SUBNET=$(…)** : stocke l’ID du subnet dans la variable `PUB_SUBNET` pour les étapes suivantes

### Création du Subnet Privé (pour l'instance cachée)

La commande suivante crée un **subnet privé** dans le VPC et récupère son identifiant.

```Bash
PRIV_SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone eu-west-3a --query 'Subnet.SubnetId' --output text)
```

![image](https://hackmd.io/_uploads/r1P73ve5-e.png)

![image](https://hackmd.io/_uploads/SyBrhwlq-e.png)

#### Explication

- **aws ec2 create-subnet** : crée un subnet dans un VPC existant
- **--vpc-id $VPC_ID** : identifiant du VPC où sera créé le subnet
- **--cidr-block 10.0.2.0/24** : plage d’adresses IP privées du subnet privé
- **--availability-zone eu-west-3a** : zone de disponibilité AWS où le subnet sera situé
- **--query 'Subnet.SubnetId'** : récupère uniquement l’identifiant du subnet
- **--output text** : formate la sortie en texte brut pour l’utiliser dans une variable shell
- **PRIV_SUBNET=$(…)** : stocke l’ID du subnet dans la variable `PRIV_SUBNET` pour les étapes suivantes

### Configuration Internet (IGW + Route)

La séquence suivante crée un **Internet Gateway (IGW)**, l’attache au VPC, et configure la **route par défaut** pour le trafic Internet.

```bash
# Créer l'Internet Gateway et récupérer son ID
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)
```

![image](https://hackmd.io/_uploads/SJ_Dnwx5We.png)

![image](https://hackmd.io/_uploads/r1iFhvg9bl.png)

```Bash
# Attacher l'Internet Gateway au VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
```

![image](https://hackmd.io/_uploads/r12j2Dg9Wx.png)

![image](https://hackmd.io/_uploads/H11phwx9-g.png)

```Bash
# Récupérer l'ID de la table de routage du VPC
RTB_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "RouteTables[0].RouteTableId" \
  --output text)
```

![image](https://hackmd.io/_uploads/rJoJaPx9bl.png)

```Bash
# Créer une route par défaut vers l'Internet Gateway
aws ec2 create-route \
  --route-table-id $RTB_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID
```

![image](https://hackmd.io/_uploads/rkUWpwgcWl.png)

![image](https://hackmd.io/_uploads/r1PBTwg9Zx.png)

#### Explication

- **aws ec2 create-internet-gateway** : crée une passerelle Internet pour permettre l’accès à Internet depuis le VPC
- **--query 'InternetGateway.InternetGatewayId'** : récupère uniquement l’ID de l’IGW
- **--output text** : formate la sortie en texte brut pour l’utiliser dans une variable
- **aws ec2 attach-internet-gateway** : attache l’IGW au VPC spécifié
- **aws ec2 describe-route-tables** : récupère les tables de routage associées au VPC
- **aws ec2 create-route** : crée une route par défaut vers l’IGW pour tout le trafic sortant (`0.0.0.0/0`)

**Variables** :

- `IGW_ID` → ID de l’Internet Gateway
- `RTB_ID` → ID de la table de routage du VPC

### Activation IP publique automatique sur le subnet public

La commande suivante configure le **subnet public** pour que toutes les nouvelles instances lancées reçoivent automatiquement une adresse IP publique.

```Bash
aws ec2 modify-subnet-attribute --subnet-id $PUB_SUBNET --map-public-ip-on-launch
```

![image](https://hackmd.io/_uploads/rJIcpwx5Zx.png)

Vérification :
![image](https://hackmd.io/_uploads/SJPXRvx9Wg.png)

#### Explication

- **aws ec2 modify-subnet-attribute** : modifie les attributs d’un subnet existant
- **--subnet-id $PUB_SUBNET** : identifiant du subnet à modifier
- **--map-public-ip-on-launch** : active l’attribution automatique d’une IP publique pour toutes les instances lancées dans ce subnet

### configuration des Security Groups pour Bastion et Instances Privées

#### Security Group pour le Bastion

```bash
# Créer le Security Group du Bastion
aws ec2 create-security-group \
  --group-name BastionSG \
  --description "SG for Bastion" \
  --vpc-id $VPC_ID

# Récupérer l'ID du SG
BASTION_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=BastionSG" \
  --query "SecurityGroups[0].GroupId" \
  --output text)

# Autoriser l'accès SSH depuis Internet
aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

![image](https://hackmd.io/_uploads/r1PLCwg5Zx.png)

![image](https://hackmd.io/_uploads/BkdORDe5Ze.png)

![image](https://hackmd.io/_uploads/BJeRc0vxqbl.png)
![image](https://hackmd.io/_uploads/Ske300wg9Ze.png)

#### Security Group pour l’Instance Privée

```Bash
# Créer le Security Group de l'instance privée
aws ec2 create-security-group \
  --group-name PrivateSG \
  --description "SG for Private" \
  --vpc-id $VPC_ID

# Récupérer l'ID du SG
PRIV_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=PrivateSG" \
  --query "SecurityGroups[0].GroupId" \
  --output text)

# Autoriser l'accès SSH uniquement depuis le Bastion
aws ec2 authorize-security-group-ingress \
  --group-id $PRIV_SG_ID \
  --protocol tcp \
  --port 22 \
  --source-group $BASTION_SG_ID
```

![image](https://hackmd.io/_uploads/HJDG1_xcWg.png)
![image](https://hackmd.io/_uploads/Hk-VyOg5bx.png)
![image](https://hackmd.io/_uploads/SJ7Ukdgc-e.png)
![image](https://hackmd.io/_uploads/SkBtydl9Wl.png)

#### Explication

- **aws ec2 create-security-group** : crée un Security Group dans le VPC
- **--group-name / --description / --vpc-id** : nom, description et VPC d’attache du SG
- **aws ec2 describe-security-groups** : récupère l’ID du Security Group créé
- **aws ec2 authorize-security-group-ingress** : définit les règles d’accès entrant
  - Pour le Bastion : SSH depuis n’importe quelle IP (`0.0.0.0/0`)
  - Pour l’instance privée : SSH uniquement depuis le Bastion (via son SG_ID)

**Variables** :

- `BASTION_SG_ID` → ID du SG du Bastion
- `PRIV_SG_ID` → ID du SG de l’instance privée

### Network ACL

La séquence suivante crée une **Network ACL (NACL)** pour contrôler le trafic au niveau du subnet et définit des règles d’accès.

```bash
# Créer la NACL et récupérer son ID
NACL_ID=$(aws ec2 create-network-acl \
  --vpc-id $VPC_ID \
  --query 'NetworkAcl.NetworkAclId' \
  --output text)

# Règle 100 : Autoriser HTTP entrant
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --ingress \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

# Règle 200 : Deny All entrant
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --ingress \
  --rule-number 200 \
  --protocol -1 \
  --cidr-block 0.0.0.0/0 \
  --rule-action deny
```

![image](https://hackmd.io/_uploads/ry5h1_lcWl.png)
![image](https://hackmd.io/_uploads/SJmcz_e9Wl.png)

Vérification

```Bash
aws ec2 describe-network-acls \
  --network-acl-ids $NACL_ID \
  --query 'NetworkAcls[0].Entries | sort_by(@, &RuleNumber)' \
  --output table
```

![image](https://hackmd.io/_uploads/Hy8NXdxcbe.png)
![image](https://hackmd.io/_uploads/HJurXuxq-e.png)

#### Explication

- **aws ec2 create-network-acl** : crée une NACL dans le VPC
- **--query 'NetworkAcl.NetworkAclId'** : récupère uniquement l’ID de la NACL
- **--output text** : formate la sortie en texte brut pour l’utiliser dans une variable
- **aws ec2 create-network-acl-entry** : ajoute une règle à la NACL
  - **--rule-number** : numéro de la règle (priorité)
  - **--protocol** : protocole réseau (`tcp` pour HTTP, `-1` pour tout)
  - **--port-range** : plage de ports à autoriser
  - **--egress false** : règle pour le trafic entrant
  - **--rule-action allow/deny** : autorise ou bloque le trafic

**NACL_ID** → ID de la NACL créée

### Création de la clé

La commande suivante crée une **clé SSH** pour se connecter aux instances et sécurise le fichier `.pem`.

```bash
# Créer la paire de clés et sauvegarder la clé privée dans un fichier
aws ec2 create-key-pair \
  --key-name DevOpsKeyTP \
  --query 'KeyMaterial' \
  --output text > DevOpsKeyTP.pem

# Sécuriser le fichier pour SSH
chmod 400 DevOpsKeyTP.pem
```

![image](https://hackmd.io/_uploads/BJKYX_ecZx.png)
![image](https://hackmd.io/_uploads/B18pm_xqWl.png)

#### Explication

- **aws ec2 create-key-pair** : crée une paire de clés SSH dans AWS
- **--key-name DevOpsKeyTP** : nom de la clé
- **--query 'KeyMaterial'** : récupère uniquement la clé privée
- **--output text > DevOpsKeyTP.pem** : enregistre la clé privée dans un fichier `.pem`
- **chmod 400 DevOpsKeyTP.pem** : protège le fichier en lecture seule pour l’utilisateur (obligatoire pour SSH)

### Lancement des instances

#### 1. Le Bastion (Subnet Public)

```bash
aws ec2 run-instances \
  --image-id ami-05d43d5e94bb6eb95 \
  --count 1 \
  --instance-type t2.micro \
  --key-name DevOpsKeyTP \
  --security-group-ids $BASTION_SG_ID \
  --subnet-id $PUB_SUBNET \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=BastionHost}]'
```

![image](https://hackmd.io/_uploads/Hy4U4_g5-l.png)

![image](https://hackmd.io/_uploads/B17FNOe9Wx.png)

#### 2. L'Instance Privée (Subnet Privé, sans IP publique)

```bash
aws ec2 run-instances \
  --image-id ami-05d43d5e94bb6eb95 \
  --count 1 \
  --instance-type t2.micro \
  --key-name DevOpsKeyTP \
  --security-group-ids $PRIV_SG_ID \
  --subnet-id $PRIV_SUBNET \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=PrivateServer}]'
```

![image](https://hackmd.io/_uploads/ByR2Vdl9Wg.png)
![image](https://hackmd.io/_uploads/HkfZBOl9be.png)

#### Explication

- **aws ec2 run-instances** : lance une ou plusieurs instances EC2
- **--image-id ami-…** : AMI utilisée pour créer l’instance
- **--count 1** : nombre d’instances à lancer
- **--instance-type t2.micro** : type d’instance (ressources CPU/mémoire)
- **--key-name DevOpsKeyTP** : clé SSH pour se connecter
- **--security-group-ids** : groupe de sécurité associé à l’instance
- **--subnet-id** : subnet dans lequel l’instance sera déployée
- **--tag-specifications** : ajoute un tag pour identifier l’instance (`BastionHost` ou `PrivateServer`)

Le **Bastion** reçoit une IP publique automatiquement, l’**Instance Privée** n’a pas d’IP publique.

### Test de connexion (Jump Host)

Attends 1 minute que les instances démarrent, puis récupère les adresses IP :

```bash
# Récupérer l'IP publique du Bastion
IP_BASTION=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=BastionHost" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

# Récupérer l'IP privée de l'instance privée
IP_PRIVEE=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=PrivateServer" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].PrivateIpAddress" \
  --output text)

# Afficher les IPs
echo "Bastion IP: $IP_BASTION"
echo "Private IP: $IP_PRIVEE"
```

![image](https://hackmd.io/_uploads/rJLNHulcWe.png)

#### Explication

- **aws ec2 describe-instances** : récupère des informations sur les instances EC2
- **--filters "Name=tag:Name,Values=…"** : sélectionne l’instance selon son tag (`BastionHost` ou `PrivateServer`)
- **"Name=instance-state-name,Values=running"** : s’assure que l’instance est en état **running**
- **--query …** : récupère uniquement l’adresse IP (publique pour le Bastion, privée pour l’instance privée)
- **--output text** : formate la sortie pour l’utiliser directement dans une variable shell
- **echo** : affiche les adresses IP pour vérifier la connectivité

### Connexion finale via Bastion

La commande suivante permet de se connecter à l’**instance privée** depuis ton PC en passant par le **Bastion** :

```bash
ssh -i DevOpsKeyTP.pem -J ec2-user@$IP_BASTION ec2-user@$IP_PRIVEE
```

![image](https://hackmd.io/_uploads/SkH_UOxqZl.png)

#### Explication

- **ssh** : commande pour se connecter à distance à une instance Linux
- **-i DevOpsKeyTP.pem** : utilise la clé SSH privée pour l’authentification
- **-J ec2-user@$IP_BASTION** : spécifie le Bastion comme Jump Host (point de passage sécurisé)
- **ec2-user@$IP_PRIVEE** : se connecte à l’instance privée depuis le Bastion

### Netoyage

```Bash
echo "--- DÉBUT DU NETTOYAGE COMPLET ---"

# 1. Récupération des IDs
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=cidr-block,Values=10.0.0.0/16" --query "Vpcs[0].VpcId" --output text)

if [ "$VPC_ID" == "None" ] || [ -z "$VPC_ID" ]; then
    echo "Erreur : VPC non trouvé. Fin du script."
    exit 1
fi

BASTION_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=BastionHost" "Name=instance-state-name,Values=running,stopped,pending" --query "Reservations[0].Instances[0].InstanceId" --output text)
PRIVATE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=PrivateServer" "Name=instance-state-name,Values=running,stopped,pending" --query "Reservations[0].Instances[0].InstanceId" --output text)

# 2. Suppression des Instances avec attente
echo "Terminaison des instances..."
# On vérifie si les IDs existent avant de tenter la suppression
[ "$BASTION_ID" != "None" ] && aws ec2 terminate-instances --instance-ids $BASTION_ID > /dev/null
[ "$PRIVATE_ID" != "None" ] && aws ec2 terminate-instances --instance-ids $PRIVATE_ID > /dev/null

if [ "$BASTION_ID" != "None" ] || [ "$PRIVATE_ID" != "None" ]; then
    aws ec2 wait instance-terminated --instance-ids $BASTION_ID $PRIVATE_ID
    echo "Instances terminées."
fi

# Pause pour libérer les interfaces réseau (ENI)
echo "Attente de libération des ressources (15s)..."
sleep 15

# 3. Suppression de la Paire de Clés (Key Pair)
echo "Suppression de la clé DevOpsKeyTP..."
aws ec2 delete-key-pair --key-name DevOpsKeyTP
# Optionnel : supprimer le fichier local .pem s'il existe
[ -f "DevOpsKeyTP.pem" ] && rm DevOpsKeyTP.pem
echo "Clé supprimée (AWS et locale)."

# 4. Suppression des Security Groups
echo "Suppression des Security Groups..."
SG_BASTION=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=BastionSG" "Name=vpc-id,Values=$VPC_ID" --query "SecurityGroups[0].GroupId" --output text)
SG_PRIVATE=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=PrivateSG" "Name=vpc-id,Values=$VPC_ID" --query "SecurityGroups[0].GroupId" --output text)

[ "$SG_PRIVATE" != "None" ] && [ -n "$SG_PRIVATE" ] && aws ec2 delete-security-group --group-id $SG_PRIVATE
[ "$SG_BASTION" != "None" ] && [ -n "$SG_BASTION" ] && aws ec2 delete-security-group --group-id $SG_BASTION
echo "Security Groups supprimés."

# 5. Détacher et supprimer l'Internet Gateway
echo "Suppression de l'Internet Gateway..."
IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query "InternetGateways[0].InternetGatewayId" --output text)
if [ "$IGW_ID" != "None" ] && [ -n "$IGW_ID" ]; then
    aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
    aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
fi

# 6. Supprimer la NACL personnalisée
echo "Suppression de la NACL..."
NACL_IDS=$(aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID" --query "NetworkAcls[?IsDefault==\`false\`].NetworkAclId" --output text)
for nacl in $NACL_IDS; do
    [ "$nacl" != "None" ] && aws ec2 delete-network-acl --network-acl-id $nacl
done

# 7. Supprimer les Subnets
echo "Suppression des Subnets..."
for subnet in $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[*].SubnetId" --output text); do
    aws ec2 delete-subnet --subnet-id $subnet
done

# 8. Suppression finale du VPC
echo "Suppression du VPC..."
aws ec2 delete-vpc --vpc-id $VPC_ID
echo "NETTOYAGE TERMINÉ AVEC SUCCÈS !"
```

### Vérification finale

```Bash
echo "--- VÉRIFICATION DU NETTOYAGE ---"

echo "1. Instances (doit être vide ou 'terminated') :"
aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,State.Name]" --output table

echo "2. VPCs (ne doit rester que le 'Default') :"
aws ec2 describe-vpcs --query "Vpcs[*].[VpcId,CidrBlock,IsDefault]" --output table

echo "3. Sécurity Groups (ne doit rester que les 'default') :"
aws ec2 describe-security-groups --query "SecurityGroups[*].[GroupId,GroupName]" --output table

echo "4. Internet Gateways (doit être vide ou lié au VPC par défaut) :"
aws ec2 describe-internet-gateways --query "InternetGateways[*].InternetGatewayId" --output table

echo "5. Paires de clés (vérifie si DevOpsKeyTP a disparu) :"
aws ec2 describe-key-pairs --query "KeyPairs[*].KeyName" --output table
```
