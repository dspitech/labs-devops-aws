# LAB : AWS RDS MySQL Lab (CLI)

Ce laboratoire montre comment déployer une base de données **MySQL sur AWS RDS** en utilisant **AWS CLI**.

L'objectif est de comprendre :

- la configuration réseau d'une base RDS
- la gestion de la sécurité avec **Security Groups**
- la création d'une instance **MySQL managée**
- la connexion à la base avec **MySQL Workbench**

Ce lab suit une approche **Infrastructure as Code via CLI** utilisée dans les environnements **DevOps**.

---

## Objectifs

À la fin de ce lab vous saurez :

- récupérer les ressources réseau AWS existantes
- créer un **Security Group** pour une base de données
- créer un **DB Subnet Group**
- déployer une instance **Amazon RDS MySQL**
- se connecter à la base depuis **MySQL Workbench**
- nettoyer les ressources pour éviter les coûts

## Architecture du La

```YAML
MySQL Workbench
       │
       │ Port 3306
       ▼
Security Group (MySQL access)
       │
       ▼
Amazon RDS MySQL Instance
       │
       ▼
DB Subnet Group
       │
       ▼
AWS VPC (Default)
```

## Définitions importantes

### VPC (Virtual Private Cloud)

Un réseau virtuel isolé dans AWS permettant de contrôler le **routage** et la **sécurité** des ressources.

---

### Subnet

Un segment du **VPC** situé dans une **Availability Zone**.

**Amazon RDS** exige au moins **2 subnets dans des zones différentes** pour garantir la **haute disponibilité**.

---

### Security Group

Un **pare-feu virtuel** qui contrôle le trafic **entrant et sortant**.

Dans ce lab on autorise :

- **TCP 3306 → MySQL**

---

### DB Subnet Group

Un **groupe de subnets** dans lequel **RDS** peut déployer son instance.

---

### Amazon RDS

Service AWS qui permet de déployer des **bases de données managées** :

- MySQL
- PostgreSQL
- MariaDB
- etc.

**Avantages :**

- sauvegardes automatiques
- patching automatique
- haute disponibilité
- monitoring intégré

## Remarque

Ce lab doit être réalisé dans **AWS CloudShell**.

## Déploiement

### Étape 1 : Préparation de l'environnement

Nous récupérons les informations du VPC par défaut afin d'intégrer correctement RDS au réseau.

```bash
# 1. Récupération du VPC par défaut
VPC_ID=$(aws ec2 describe-vpcs \
--filters "Name=isDefault,Values=true" \
--query "Vpcs[0].VpcId" \
--output text)

# 2. Récupération de 2 subnets
SUBNET_IDS=$(aws ec2 describe-subnets \
--filters "Name=vpc-id,Values=$VPC_ID" \
--query "Subnets[:2].SubnetId" \
--output text | sed 's/\t/ /g')

echo "VPC utilisé : $VPC_ID"
echo "Subnets sélectionnés : $SUBNET_IDS"
```

![image](https://hackmd.io/_uploads/B1UWYVW5-l.png)

### Étape 2 : Configuration du Security Group

Nous créons un Security Group dédié à la base de données.

Ce groupe autorise les connexions MySQL depuis l'extérieur.

**Note** :
Dans un environnement production, il est recommandé de restreindre l'accès à une adresse IP spécifique.

```bash
# Création du Security Group
RDS_SG_ID=$(aws ec2 create-security-group \
--group-name "DevOps-RDS-SG" \
--description "Security group for RDS MySQL" \
--vpc-id $VPC_ID \
--output text)

# Autoriser MySQL (port 3306)
aws ec2 authorize-security-group-ingress \
--group-id $RDS_SG_ID \
--protocol tcp \
--port 3306 \
--cidr 0.0.0.0/0

echo "Security Group créé : $RDS_SG_ID"
```

![image](https://hackmd.io/_uploads/r1iLcEb5Zg.png)

![image](https://hackmd.io/_uploads/rygu9V-5We.png)

![image](https://hackmd.io/_uploads/SkgocVWcWg.png)

### Étape 3 : Déploiement de l'infrastructure RDS

#### 3.1 Création du DB Subnet Group

```bash
aws rds create-db-subnet-group \
--db-subnet-group-name "DevOpsSubnetGroup" \
--db-subnet-group-description "Groupe de sous-réseaux pour RDS" \
--subnet-ids $SUBNET_IDS
```

![image](https://hackmd.io/_uploads/H17C9V-qWg.png)

#### 3.2 Création de l'instance MySQL

```bash
aws rds create-db-instance \
--db-instance-identifier DevOpsDB \
--db-instance-class db.t3.micro \
--engine mysql \
--master-username admin \
--master-user-password Admin1234 \
--allocated-storage 20 \
--vpc-security-group-ids $RDS_SG_ID \
--db-subnet-group-name DevOpsSubnetGroup \
--publicly-accessible \
--backup-retention-period 0 \
--no-multi-az \
--no-deletion-protection
```

![image](https://hackmd.io/_uploads/S1CoiVZ5-g.png)

![image](https://hackmd.io/_uploads/rkCniVWcWx.png)

![image](https://hackmd.io/_uploads/rksZh4b9-g.png)

![image](https://hackmd.io/_uploads/SycshE-q-l.png)

##### Explication des paramètres

| Paramètre               | Description               |
| ----------------------- | ------------------------- |
| db-instance-class       | type de machine           |
| engine                  | moteur de base de données |
| allocated-storage       | stockage initial          |
| publicly-accessible     | accès externe autorisé    |
| multi-az                | haute disponibilité       |
| backup-retention-period | durée de sauvegarde       |

### Étape 4 : Vérification du statut

Le démarrage de l'instance prend 5 à 8 minutes.

Commande pour vérifier

```bash
aws rds describe-db-instances \
--db-instance-identifier DevOpsDB \
--query "DBInstances[0].[DBInstanceStatus,Endpoint.Address]" \
--output table
```

![image](https://hackmd.io/_uploads/rkIphN-5Wg.png)

### Étape 5 : Connexion avec MySQL Workbench

### Paramètres de connexion

| Paramètre | Valeur       |
| --------- | ------------ |
| Host      | Endpoint RDS |
| Port      | 3306         |
| Username  | admin        |
| Password  | Admin1234    |

![image](https://hackmd.io/_uploads/Sk1JTVW5-x.png)

![image](https://hackmd.io/_uploads/H1p7aN-c-x.png)
![image](https://hackmd.io/_uploads/rkg8aVZ9Zg.png)

### Etape 6 : Création d'une base et de tables

Exécuter les requêtes suivantes dans MySQL Workbench.

```SQL
CREATE DATABASE DevOpsLab;

USE DevOpsLab;

CREATE TABLE Inventory (
    id INT AUTO_INCREMENT PRIMARY KEY,
    item_name VARCHAR(255) NOT NULL,
    quantity INT DEFAULT 0
);

INSERT INTO Inventory (item_name, quantity)
VALUES
('Serveur EC2', 5),
('S3 Bucket', 12);

SELECT * FROM Inventory;
```

![image](https://hackmd.io/_uploads/H1X_p4Wq-g.png)

![image](https://hackmd.io/_uploads/S1ycpN-9We.png)

### Étape 7 : Nettoyage des ressources

Important pour éviter les coûts AWS.

#### Suppression de l'instance

```Bash
aws rds delete-db-instance \
--db-instance-identifier DevOpsDB \
--skip-final-snapshot \
--delete-automated-backups
```

![image](https://hackmd.io/_uploads/rJ-yREW5be.png)

![image](https://hackmd.io/_uploads/BJeWCNZ5-g.png)

![image](https://hackmd.io/_uploads/SygoR4WqWx.png)

Une fois l'instance supprimée :

#### Supprimer le subnet-group

```bash
aws rds delete-db-subnet-group \
--db-subnet-group-name DevOpsSubnetGroup
```

![image](https://hackmd.io/_uploads/Hy8PkHWcWg.png)

![image](https://hackmd.io/_uploads/HJjjJSW9Zg.png)

```bash
aws ec2 delete-security-group \
--group-id $RDS_SG_ID
```

![image](https://hackmd.io/_uploads/ByktySZq-e.png)

![image](https://hackmd.io/_uploads/SyYCySZ9We.png)

Pour vérifier que tout a bien été supprimé (RDS, Subnet Group, Security Group), tu peux utiliser les commandes CLI suivantes.

#### Vérifier si l’instance RDS a été supprimée

```Bash
aws rds describe-db-instances \
--db-instance-identifier DevOpsDB \
--query "DBInstances[0].DBInstanceStatus" \
--output text
```

![image](https://hackmd.io/_uploads/HJoDxHW5Wl.png)

#### Vérifier si le DB Subnet Group a été supprimé

```Bash
aws rds describe-db-subnet-groups \
--db-subnet-group-name DevOpsSubnetGroup \
--query "DBSubnetGroups[0].DBSubnetGroupName" \
--output text
```

![image](https://hackmd.io/_uploads/ByOjeSbq-e.png)

#### Vérifier si le Security Group a été supprimé

```Bash
aws ec2 describe-security-groups \
--group-ids $RDS_SG_ID \
--query "SecurityGroups[0].GroupId" \
--output text
```

![image](https://hackmd.io/_uploads/HJLe-BZcbl.png)

### Résumé

Dans ce lab nous avons :

- ✔ configuré le réseau AWS
- ✔ créé un Security Group
- ✔ déployé une instance Amazon RDS MySQL
- ✔ connecté MySQL Workbench
- ✔ créé une base et une table
- ✔ nettoyé les ressources

---

### Compétences démontrées

- AWS CLI
- Amazon RDS
- Networking AWS
- Security Groups
- SQL
- DevOps Infrastructure Setup
