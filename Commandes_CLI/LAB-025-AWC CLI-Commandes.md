# AWC CLI - Commandes

## CloudShell / CLI Utilities

```Bash
# =========================
# AWS CLI - CONFIGURATION
# =========================

# Configure l’AWS CLI avec ton Access Key, Secret Key et région par défaut
# Exécute cette commande et suis les invites
aws configure

# Affiche la configuration actuelle de l’AWS CLI
aws configure list

# Affiche le contenu détaillé (profiles, variables d'environnement, etc.)
aws configure list-profiles


# =========================
# AWS STS - IDENTITE DU COMPTE
# =========================

# Vérifie l’identité du compte et de l’utilisateur courant
# Retourne : Account, ARN, UserId
aws sts get-caller-identity


# =========================
# AWS EC2 - LISTE DES REGIONS
# =========================

# Liste toutes les régions disponibles pour EC2
aws ec2 describe-regions \
  --all-regions

# Exemple de filtrage par région spécifique
aws ec2 describe-regions \
  --region ap-south-1
```

## EC2 - Compute Instances

```Bash
# Liste toutes les instances EC2 avec leurs détails (ID, état, IP, etc.)
aws ec2 describe-instances

# Lance une nouvelle instance EC2
# --image-id : ID de l'AMI (image système)
# --instance-type : type de machine (CPU/RAM)
# --key-name : nom de la clé SSH pour se connecter
# --security-group-ids : groupe de sécurité (pare-feu)
aws ec2 run-instances --image-id ami-12345 --instance-type t3.micro --key-name my-key --security-group-ids sg-12345

# Arrête une instance EC2 (sans la supprimer)
# L'instance peut être redémarrée plus tard
aws ec2 stop-instances --instance-ids i-12345

# Démarre une instance EC2 arrêtée
aws ec2 start-instances --instance-ids i-12345

# Supprime définitivement une instance EC2
# ATTENTION : action irréversible
aws ec2 terminate-instances --instance-ids i-12345

# Affiche le statut des instances (running, stopped, etc.)
aws ec2 describe-instance-status
```

## EBS - Elastic Block Store

```Bash
# Liste tous les volumes EBS (Elastic Block Store)
# Donne des infos comme taille, zone, état, etc.
aws ec2 describe-volumes

# Crée un nouveau volume EBS
# --size : taille en Go
# --availability-zone : zone où sera créé le volume
# --volume-type : type de stockage (gp3, io1, etc.)
aws ec2 create-volume --size 20 --availability-zone us-east-1a --volume-type gp3

# Attache un volume EBS à une instance EC2
# --volume-id : ID du volume
# --instance-id : ID de l'instance
# --device : point de montage (ex: /dev/sdf)
aws ec2 attach-volume --volume-id vol-12345 --instance-id i-12345 --device /dev/sdf

# Crée un snapshot (sauvegarde) d’un volume
# --volume-id : volume à sauvegarder
# --description : description du snapshot
aws ec2 create-snapshot --volume-id vol-12345 --description "backup"

# Crée un nouveau volume à partir d’un snapshot existant
# --snapshot-id : ID du snapshot
# --availability-zone : zone de création
aws ec2 create-volume --snapshot-id snap-12345 --availability-zone us-east-1a
```

## S3 - Simple Storage Service

```Bash
# Liste le contenu des buckets S3 (ou des objets dans un bucket)
# "ls" = list (et non "1s")
aws s3 ls

# Crée un bucket S3
# Remplace "mybucket" par un nom unique globalement
aws s3 mb s3://mybucket

# Copie un fichier local vers un bucket S3
# file.txt = fichier source
# s3://mybucket/ = destination
aws s3 cp file.txt s3://mybucket/

# Synchronise un dossier local avec un dossier S3
# ./data = dossier local
# s3://mybucket/data = destination dans le bucket
aws s3 sync ./data s3://mybucket/data

# Supprime un bucket S3
# --force : supprime aussi tout le contenu du bucket
aws s3 rb s3://mybucket --force
```

## IAM — Identity & Access Management

```Bash
# Liste tous les utilisateurs IAM
aws iam list-users

# Crée un nouvel utilisateur IAM nommé "devops"
# Attention : il manquait un espace dans ta commande
aws iam create-user --user-name devops

# Attache une politique (policy) à l'utilisateur
# Ici : donne un accès administrateur complet (très permissif)
# --policy-arn : ARN de la policy AWS
aws iam attach-user-policy \
  --user-name devops \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Crée un rôle IAM
# --role-name : nom du rôle
# --assume-role-policy-document : fichier JSON définissant qui peut assumer ce rôle
aws iam create-role \
  --role-name s3-access \
  --assume-role-policy-document file://trust-policy.json

# Affiche les informations sur l'identité AWS utilisée (compte, utilisateur, ARN)
aws sts get-caller-identity
```

## KMS — Key Management Service

```Bash
# Liste toutes les clés KMS (Key Management Service)
# Affiche les IDs des clés disponibles
aws kms list-keys

# Crée une nouvelle clé KMS
# --description : description de la clé
aws kms create-key --description "EBS Encryption Key"

# Affiche les détails d’une clé KMS
# --key-id : peut être un ID de clé, un ARN ou un alias
# Exemple : alias/mykey
aws kms describe-key --key-id alias/mykey

# Active la rotation automatique de la clé (recommandé pour la sécurité)
# Remplacer <key-id> par l’ID réel de la clé
aws kms enable-key-rotation --key-id <key-id>
```

## ECS - Elastic Container Service

```Bash
#!/bin/bash

# Liste tous les clusters ECS (Elastic Container Service)
aws ecs list-clusters

# Liste les services dans un cluster spécifique
# --cluster : nom ou ARN du cluster
aws ecs list-services --cluster my-cluster

# Décrit les tâches (tasks) dans un cluster
# Nécessite généralement l'ID des tâches (--tasks)
# Sinon, utilise d'abord "list-tasks"
aws ecs describe-tasks --cluster my-cluster --tasks task-id-12345

# Met à jour un service ECS
# --desired-count : nombre de tâches souhaitées (scaling)
# Correction : "desiredcount" → "desired-count"
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --desired-count 2
```

## EKS - Elastic Kubernetes Service

```Bash
# Liste tous les clusters EKS (Elastic Kubernetes Service)
aws eks list-clusters

# Affiche les détails d’un cluster EKS
# --name : nom du cluster
# Correction : espace manquant après "describe-cluster"
aws eks describe-cluster --name my-eks-cluster

# Met à jour le kubeconfig pour se connecter au cluster avec kubectl
# Correction : espace manquant après "update-kubeconfig"
# Cela ajoute la config dans ~/.kube/config
aws eks update-kubeconfig --name my-eks-cluster
```

## Lambda - Serverless Functions

```Bash
# Liste toutes les fonctions Lambda
aws lambda list-functions

# Crée une nouvelle fonction Lambda
# --function-name : nom de la fonction
# --runtime : environnement (ex: python3.9)
# --role : ARN du rôle IAM associé à la fonction
# --handler : point d’entrée du code (fichier.fonction)
# --zip-file : archive contenant le code
aws lambda create-function \
  --function-name MyFunc \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/my-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip

# Exécute (invoke) une fonction Lambda
# Le résultat est sauvegardé dans output.json
aws lambda invoke --function-name MyFunc output.json

# Met à jour le code de la fonction Lambda
# Correction : le chemin du fichier ZIP était coupé
aws lambda update-function-code \
  --function-name MyFunc \
  --zip-file fileb://function.zip
```

## CloudWatch - Monitoring & Logs

```Bash
# Liste toutes les métriques disponibles dans CloudWatch
aws cloudwatch list-metrics

# Récupère des statistiques sur une métrique (ex: CPU EC2)
# --namespace : service AWS (ex: AWS/EC2)
# --metric-name : nom de la métrique
# --dimensions : filtre (ex: InstanceId)
# --start-time / --end-time : période d'analyse
# --period : intervalle en secondes
# --statistics : type de statistique (Average, Sum, etc.)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-12345 \
  --start-time 2025-12-04T00:00:00Z \
  --end-time 2025-12-04T23:59:59Z \
  --period 300 \
  --statistics Average

# Liste tous les groupes de logs CloudWatch Logs
aws logs describe-log-groups

# Récupère les événements d’un log stream
# --log-group-name : groupe de logs (ex: Lambda)
# --log-stream-name : nom du flux de logs
# Correction : "log-streamname" → "log-stream-name"
aws logs get-log-events \
  --log-group-name /aws/lambda/myfunc \
  --log-stream-name latest
```

## CloudTrail - Audit Logs

```Bash
# Liste tous les trails CloudTrail (journalisation des actions AWS)
aws cloudtrail describe-trails

# Crée un nouveau trail CloudTrail
# --name : nom du trail
# --s3-bucket-name : bucket S3 où seront stockés les logs
aws cloudtrail create-trail \
  --name mytrail \
  --s3-bucket-name mybucket

# Démarre la journalisation pour le trail
# (sinon le trail existe mais ne collecte pas encore de logs)
aws cloudtrail start-logging --name mytrail
```

## SSM - Systems Manager

```Bash
# Liste toutes les instances gérées par AWS SSM
aws ssm describe-instance-information

# Exécute une commande sur une ou plusieurs instances gérées par SSM
# --instance-ids : ID des instances ciblées
# --document-name : nom du document SSM (ex: AWSRunShellScript pour scripts shell)
# --parameters : commandes à exécuter (ici: afficher l'espace disque)
aws ssm send-command \
  --instance-ids i-12345 \
  --document-name "AWSRunShellScript" \
  --parameters 'commands=["df -h"]'

# Démarre une session interactive avec une instance gérée par SSM
# Remplace i-12345 par l'ID de ton instance
aws ssm start-session --target i-12345
```

## Route 53 - DNS & Domains

```Bash
# Liste toutes les zones hébergées dans Route 53
aws route53 list-hosted-zones

# Liste tous les enregistrements DNS d'une zone spécifique
# --hosted-zone-id : ID de la zone hébergée (ex: Z123456789)
aws route53 list-resource-record-sets --hosted-zone-id Z123456789

# Modifie les enregistrements DNS d'une zone
# --change-batch : fichier JSON contenant les modifications
# Exemple de fichier JSON "record.json" :
# {
#   "Changes": [
#     {
#       "Action": "UPSERT",
#       "ResourceRecordSet": {
#         "Name": "example.mydomain.com",
#         "Type": "A",
#         "TTL": 300,
#         "ResourceRecords": [{"Value": "192.0.2.1"}]
#       }
#     }
#   ]
# }
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch file://record.json
```

## VPC- Networking

```Bash
# Liste tous les VPC existants dans le compte
aws ec2 describe-vpcs

# Crée un nouveau VPC
# --cidr-block : plage d'adresses IP pour le VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Crée un sous-réseau (subnet) dans un VPC spécifique
# --vpc-id : ID du VPC parent
# --cidr-block : plage d'adresses IP pour le subnet
aws ec2 create-subnet --vpc-id vpc-12345 --cidr-block 10.0.1.0/24

# Liste tous les groupes de sécurité existants
aws ec2 describe-security-groups
```

## Auto Scaling

```Bash
# Liste tous les groupes Auto Scaling existants
aws autoscaling describe-auto-scaling-groups

# Modifie le nombre de instances souhaitées (desired capacity) dans un Auto Scaling Group
# --auto-scaling-group-name : nom du groupe à modifier
# --desired-capacity : nombre d'instances souhaité
# Correction : suppression du tiret supplémentaire
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name my-asg \
  --desired-capacity 3
```

## CloudFront - CDN

```Bash
# Liste toutes les distributions CloudFront existantes
aws cloudfront list-distributions

# Crée une nouvelle distribution CloudFront
# --origin-domain-name : définit le domaine d'origine (ici un bucket S3)
aws cloudfront create-distribution \
  --origin-domain-name mybucket.s3.amazonaws.com

# Crée une invalidation de cache pour une distribution CloudFront
# --distribution-id : identifiant de la distribution (à remplacer par le tien)
# --paths "/*" : invalide tous les fichiers du cache
aws cloudfront create-invalidation \
  --distribution-id E12345 \
  --paths "/*"
```

## RDS - Databases

```Bash
# Liste toutes les instances RDS existantes
aws rds describe-db-instances

# Crée une nouvelle instance RDS
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password MyPass123 \
  --allocated-storage 20

# Vérifie le statut de l'instance (utile après création)
# Permet de voir si elle est "creating", "available", etc.
aws rds describe-db-instances \
  --db-instance-identifier mydb

# Démarre une instance RDS (si elle est arrêtée)
aws rds start-db-instance \
  --db-instance-identifier mydb

# Arrête une instance RDS (pour économiser des coûts)
aws rds stop-db-instance \
  --db-instance-identifier mydb

# Redémarre l'instance RDS
aws rds reboot-db-instance \
  --db-instance-identifier mydb

# Modifie la taille de l'instance (scaling vertical)
# --apply-immediately : applique le changement sans attendre la maintenance window
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.small \
  --apply-immediately

# Crée un snapshot (sauvegarde manuelle)
aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-snapshot-1

# Liste les snapshots disponibles
aws rds describe-db-snapshots \
  --db-instance-identifier mydb

# Supprime une instance RDS
# --skip-final-snapshot : supprime sans sauvegarde finale
aws rds delete-db-instance \
  --db-instance-identifier mydb \
  --skip-final-snapshot

# Supprime un snapshot
aws rds delete-db-snapshot \
  --db-snapshot-identifier mydb-snapshot-1
```

## DynamoDB - NOSQL

```Bash
# Liste toutes les tables DynamoDB
aws dynamodb list-tables

# Scanne une table (lit toutes les données → coûteux sur grosses tables)
# --table-name : nom de la table
aws dynamodb scan \
  --table-name MyTable

# Ajoute un item dans la table
# Format JSON requis avec types DynamoDB (S = String, N = Number, etc.)
aws dynamodb put-item \
  --table-name MyTable \
  --item '{
    "id": {"S":"1"},
    "name": {"S":"Test"}
  }'

# Récupère un item spécifique (plus efficace que scan)
# --key : clé primaire de l'item
aws dynamodb get-item \
  --table-name MyTable \
  --key '{
    "id": {"S":"1"}
  }'

# Met à jour un attribut d'un item existant
aws dynamodb update-item \
  --table-name MyTable \
  --key '{"id":{"S":"1"}}' \
  --update-expression "SET #n = :newname" \
  --expression-attribute-names '{"#n":"name"}' \
  --expression-attribute-values '{":newname":{"S":"UpdatedName"}}'

# Supprime un item
aws dynamodb delete-item \
  --table-name MyTable \
  --key '{
    "id": {"S":"1"}
  }'

# Crée une table DynamoDB
# --attribute-definitions : définition des attributs
# --key-schema : clé primaire
# --billing-mode PAY_PER_REQUEST : facturation à la demande (serverless)
aws dynamodb create-table \
  --table-name MyTable \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Supprime une table
aws dynamodb delete-table \
  --table-name MyTable
```

## ECR - Elastic Container Registry

```Bash
# Crée un repository ECR (Elastic Container Registry)
# --repository-name : nom du dépôt Docker
aws ecr create-repository \
  --repository-name myapp

# Authentifie Docker auprès d’ECR
# get-login-password : récupère un token temporaire
# docker login : connexion au registry AWS
aws ecr get-login-password \
  --region ap-south-1 | docker login \
  --username AWS \
  --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com

# Construit une image Docker localement
# -t : tag de l'image (nom:version)
docker build -t myapp:latest .

# Tag l'image pour ECR (obligatoire avant push)
# Format : <account-id>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>
docker tag myapp:latest \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:latest

# Push l'image vers ECR
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:latest

# Liste les repositories ECR
aws ecr describe-repositories

# Liste les images dans un repository
aws ecr list-images \
  --repository-name myapp

# Supprime une image spécifique
aws ecr batch-delete-image \
  --repository-name myapp \
  --image-ids imageTag=latest

# Supprime un repository
# --force : supprime même s’il contient des images
aws ecr delete-repository \
  --repository-name myapp \
  --force
```

## WAF / Shield

```Bash
# =========================
# AWS WAF - LIST & INFO
# =========================

# Liste les Web ACLs
aws wafv2 list-web-acls \
  --scope REGIONAL

# Détails d’un Web ACL
aws wafv2 get-web-acl \
  --name MyWebACL \
  --scope REGIONAL \
  --id abc123

# Liste des règles managées AWS disponibles
aws wafv2 list-available-managed-rule-groups \
  --scope REGIONAL


# =========================
# AWS WAF - CRÉATION
# =========================

# Crée un Web ACL simple (sans règles)
aws wafv2 create-web-acl \
  --name MyWebACL \
  --scope REGIONAL \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=MyWebACL \
  --rules '[]'

# Crée un Web ACL avec une règle managée AWS
aws wafv2 create-web-acl \
  --name MyWebACLWithRules \
  --scope REGIONAL \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=MyWebACL \
  --rules '[
    {
      "Name": "AWS-AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "OverrideAction": { "None": {} },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRules"
      }
    }
  ]'


# =========================
# AWS WAF - ASSOCIATION
# =========================

# Associe un Web ACL à une ressource (ex: ALB)
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:region:account-id:regional/webacl/MyWebACL/abc123 \
  --resource-arn arn:aws:elasticloadbalancing:region:account-id:loadbalancer/app/my-alb/xyz

# Désassocie un Web ACL
aws wafv2 disassociate-web-acl \
  --resource-arn arn:aws:elasticloadbalancing:region:account-id:loadbalancer/app/my-alb/xyz


# =========================
# AWS WAF - SUPPRESSION
# =========================

# Supprime un Web ACL
# (nécessite le lock token récupéré via get-web-acl)
aws wafv2 delete-web-acl \
  --name MyWebACL \
  --scope REGIONAL \
  --id abc123 \
  --lock-token TOKEN


# =========================
# AWS SHIELD - INFO
# =========================

# Liste les attaques détectées
aws shield list-attacks

# Détaille une attaque spécifique
aws shield describe-attack \
  --attack-id 12345678

# Liste les protections actives
aws shield list-protections


# =========================
# AWS SHIELD - CRÉATION
# =========================

# Active une protection Shield sur une ressource
aws shield create-protection \
  --name MyProtection \
  --resource-arn arn:aws:elasticloadbalancing:region:account-id:loadbalancer/app/my-alb/xyz


# =========================
# AWS SHIELD - SUPPRESSION
# =========================

# Supprime une protection Shield
aws shield delete-protection \
  --protection-id abc123
```

## EventBridge

```Bash
# =========================
# EVENTBRIDGE - LIST & INFO
# =========================

# Liste toutes les règles EventBridge
aws events list-rules

# Détails d’une règle spécifique
aws events describe-rule \
  --name DailyBackup


# =========================
# EVENTBRIDGE - CRÉATION
# =========================

# Crée une règle planifiée (cron / rate)
# --schedule-expression : exécution périodique
# rate(1 day) = tous les jours
aws events put-rule \
  --name DailyBackup \
  --schedule-expression "rate(1 day)" \
  --state ENABLED

# Exemple avec cron (tous les jours à minuit UTC)
aws events put-rule \
  --name DailyBackupCron \
  --schedule-expression "cron(0 0 * * ? *)" \
  --state ENABLED


# =========================
# EVENTBRIDGE - TARGET
# =========================

# Ajoute une cible à la règle (ex: Lambda)
# --targets : ressource déclenchée par la règle
aws events put-targets \
  --rule DailyBackup \
  --targets "Id"="1","Arn"="arn:aws:lambda:region:account-id:function:MyFunction"

# Liste les cibles d’une règle
aws events list-targets-by-rule \
  --rule DailyBackup


# =========================
# EVENTBRIDGE - GESTION
# =========================

# Désactive une règle
aws events disable-rule \
  --name DailyBackup

# Active une règle
aws events enable-rule \
  --name DailyBackup


# =========================
# EVENTBRIDGE - SUPPRESSION
# =========================

# Supprime les cibles (obligatoire avant suppression de règle)
aws events remove-targets \
  --rule DailyBackup \
  --ids "1"

# Supprime une règle
aws events delete-rule \
  --name DailyBackup
```

## Organizations / Control Tower

```Bash
# =========================
# ORGANIZATIONS - LIST & INFO
# =========================

# Liste tous les comptes de l’organisation
aws organizations list-accounts

# Liste les policies (SCP, Tag policies, etc.)
aws organizations list-policies \
  --filter SERVICE_CONTROL_POLICY

# Détails d’une policy spécifique
aws organizations describe-policy \
  --policy-id p-12345678

# Liste les Organizational Units (OU)
aws organizations list-organizational-units-for-parent \
  --parent-id r-xxxx

# Liste les comptes dans une OU
aws organizations list-accounts-for-parent \
  --parent-id ou-xxxx-yyyy


# =========================
# ORGANIZATIONS - CRÉATION
# =========================

# Crée une nouvelle Organizational Unit (OU)
aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name DevOU

# Crée une policy SCP (Service Control Policy)
aws organizations create-policy \
  --name DenyS3Public \
  --description "Bloque les accès publics S3" \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Deny",
        "Action": "s3:PutBucketPublicAccessBlock",
        "Resource": "*"
      }
    ]
  }'


# =========================
# ORGANIZATIONS - ASSOCIATION
# =========================

# Attache une policy à une OU ou un compte
aws organizations attach-policy \
  --policy-id p-12345678 \
  --target-id ou-xxxx-yyyy

# Liste les policies attachées à une cible
aws organizations list-policies-for-target \
  --target-id ou-xxxx-yyyy \
  --filter SERVICE_CONTROL_POLICY


# =========================
# ORGANIZATIONS - GESTION
# =========================

# Détache une policy
aws organizations detach-policy \
  --policy-id p-12345678 \
  --target-id ou-xxxx-yyyy

# Déplace un compte vers une autre OU
aws organizations move-account \
  --account-id 123456789012 \
  --source-parent-id r-xxxx \
  --destination-parent-id ou-xxxx-yyyy


# =========================
# ORGANIZATIONS - SUPPRESSION
# =========================

# Supprime une policy
aws organizations delete-policy \
  --policy-id p-12345678

# Supprime une OU (doit être vide)
aws organizations delete-organizational-unit \
  --organizational-unit-id ou-xxxx-yyyy
```

## Elasticache - Caching

```Bash
# =========================
# ELASTICACHE - LIST & INFO
# =========================

# Liste tous les clusters ElastiCache
aws elasticache describe-cache-clusters

# Détaille un cluster spécifique
aws elasticache describe-cache-clusters \
  --cache-cluster-id myredis \
  --show-cache-node-info


# =========================
# ELASTICACHE - CRÉATION
# =========================

# Crée un cluster Redis simple
# --cache-cluster-id : identifiant du cluster
# --engine : type de cache (redis ou memcached)
# --cache-node-type : type de noeud
# --num-cache-nodes : nombre de noeuds (Redis standard = 1)
aws elasticache create-cache-cluster \
  --cache-cluster-id myredis \
  --engine redis \
  --cache-node-type cache.t3.micro \
  --num-cache-nodes 1

# Optionnel : définir le paramètre de sécurité (VPC, Subnet Group, Security Group)
# --cache-subnet-group-name mysubnetgroup
# --security-group-ids sg-xxxxxx


# =========================
# ELASTICACHE - GESTION
# =========================

# Modifie le cluster (ex: changer type de noeud)
aws elasticache modify-cache-cluster \
  --cache-cluster-id myredis \
  --cache-node-type cache.t3.small \
  --apply-immediately

# Redémarre le cluster (utile après certains changements)
aws elasticache reboot-cache-cluster \
  --cache-cluster-id myredis \
  --cache-node-ids-to-reboot 0001


# =========================
# ELASTICACHE - SUPPRESSION
# =========================

# Supprime le cluster Redis
aws elasticache delete-cache-cluster \
  --cache-cluster-id myredis
```

## SNS- Notifications

```Bash
# =========================
# SNS - CRÉATION DE TOPIC
# =========================

# Crée un topic SNS
# --name : nom du topic
aws sns create-topic \
  --name alert-topic

# Récupère l'ARN du topic (nécessaire pour les abonnements et publications)
aws sns list-topics


# =========================
# SNS - ABONNEMENT
# =========================

# Abonne un email au topic
# --protocol : type de notification (email, SMS, lambda, sqs, etc.)
# --notification-endpoint : adresse de destination (email ici)
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-south-1:123456789012:alert-topic \
  --protocol email \
  --notification-endpoint you@example.com

# Remarque : le destinataire doit confirmer l'abonnement via email


# =========================
# SNS - PUBLICATION
# =========================

# Envoie un message au topic
# --message : texte du message
aws sns publish \
  --topic-arn arn:aws:sns:ap-south-1:123456789012:alert-topic \
  --message "Backup completed!"

# Exemple : message JSON (pour des notifications plus avancées)
aws sns publish \
  --topic-arn arn:aws:sns:ap-south-1:123456789012:alert-topic \
  --message '{"default": "Backup completed!", "email": "Backup finished successfully!"}' \
  --message-structure json


# =========================
# SNS - GESTION
# =========================

# Liste les abonnements d’un topic
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:ap-south-1:123456789012:alert-topic

# Supprime un abonnement
aws sns unsubscribe \
  --subscription-arn arn:aws:sns:ap-south-1:123456789012:sub-xxxxxxxxxxxx

# Supprime un topic
aws sns delete-topic \
  --topic-arn arn:aws:sns:ap-south-1:123456789012:alert-topic
```

## SQS - Messaging Queues

```Bash
# =========================
# SQS - CRÉATION DE QUEUE
# =========================

# Crée une nouvelle queue SQS
# --queue-name : nom de la queue
aws sqs create-queue \
  --queue-name myqueue

# Récupère l'URL de la queue (nécessaire pour envoyer/recevoir des messages)
aws sqs get-queue-url \
  --queue-name myqueue

# Exemple de sortie : https://sqs.ap-south-1.amazonaws.com/123456789012/myqueue
QUEUE_URL="https://sqs.ap-south-1.amazonaws.com/123456789012/myqueue"


# =========================
# SQS - ENVOI DE MESSAGE
# =========================

# Envoie un message à la queue
# --message-body : contenu du message
aws sqs send-message \
  --queue-url $QUEUE_URL \
  --message-body "Hello World"

# Envoi d’un message avec attributs personnalisés
aws sqs send-message \
  --queue-url $QUEUE_URL \
  --message-body "Order received" \
  --message-attributes '{
      "OrderId": {"DataType":"String","StringValue":"1234"},
      "Priority": {"DataType":"Number","StringValue":"1"}
  }'


# =========================
# SQS - RECEPTION DE MESSAGES
# =========================

# Récupère les messages de la queue
aws sqs receive-message \
  --queue-url $QUEUE_URL \
  --max-number-of-messages 10 \
  --wait-time-seconds 5 \
  --message-attribute-names All

# Supprime un message après traitement
# --receipt-handle : valeur obtenue depuis receive-message
aws sqs delete-message \
  --queue-url $QUEUE_URL \
  --receipt-handle RECEIPT_HANDLE


# =========================
# SQS - GESTION
# =========================

# Liste toutes les queues SQS
aws sqs list-queues

# Supprime une queue
aws sqs delete-queue \
  --queue-url $QUEUE_URL
```

## SES - Email Service

```Bash
# =========================
# SES - LISTE DES IDENTITÉS
# =========================

# Liste toutes les identités vérifiées (emails et domaines)
aws ses list-identities

# Vérifie une adresse email
# Le destinataire doit cliquer sur le lien dans l'email pour valider
aws ses verify-email-identity \
  --email-address you@example.com

# Vérifie un domaine entier (optionnel, pour envoyer depuis plusieurs adresses)
aws ses verify-domain-identity \
  --domain example.com


# =========================
# SES - ENVOI D'EMAIL
# =========================

# Envoie un email simple
aws ses send-email \
  --from you@example.com \
  --destination "ToAddresses=someone@example.com" \
  --message "Subject={Data=Test},Body={Text={Data=Hello}}"

# Exemple plus complet avec HTML et CC/BCC
aws ses send-email \
  --from you@example.com \
  --destination "ToAddresses=someone@example.com,CCAddresses=cc@example.com,BccAddresses=bcc@example.com" \
  --message '{
      "Subject": {"Data": "Test Email"},
      "Body": {
          "Text": {"Data": "Hello in plain text"},
          "Html": {"Data": "<h1>Hello in HTML</h1>"}
      }
  }'

# Liste les emails envoyés et statistiques
aws ses get-send-statistics


# =========================
# SES - GESTION
# =========================

# Supprime une identité vérifiée
aws ses delete-identity \
  --identity you@example.com
```

## CloudWatch Alarms

```Bash
# =========================
# CLOUDWATCH - CREATION D'ALARMES
# =========================

# Crée une alarme CloudWatch pour l'utilisation CPU d'une instance EC2
aws cloudwatch put-metric-alarm \
  --alarm-name CPUHigh \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-12345 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:alert-topic \
  --unit Percent

# Remarques :
# --alarm-name : nom de l'alarme
# --metric-name : métrique à surveiller (CPUUtilization)
# --namespace : namespace de la métrique (AWS/EC2 ici)
# --statistic : type de statistique (Average, Sum, Maximum…)
# --period : période de collecte des données (en secondes)
# --threshold : valeur critique qui déclenche l'alarme
# --comparison-operator : opérateur de comparaison
# --dimensions : filtre sur une instance spécifique
# --evaluation-periods : nombre de périodes consécutives dépassant le seuil avant déclenchement
# --alarm-actions : action à effectuer (ex: envoyer notification SNS)
# --unit : unité de la métrique (ici Percent)


# =========================
# CLOUDWATCH - GESTION D'ALARMES
# =========================

# Liste toutes les alarmes CloudWatch
aws cloudwatch describe-alarms

# Liste les alarmes pour une instance spécifique
aws cloudwatch describe-alarms-for-metric \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-12345

# Met à jour une alarme existante (ex: seuil 90% au lieu de 80%)
aws cloudwatch put-metric-alarm \
  --alarm-name CPUHigh \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-12345 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:alert-topic \
  --unit Percent

# Supprime une alarme
aws cloudwatch delete-alarms \
  --alarm-names CPUHigh
```

## Backup Service

```Bash
# =========================
# AWS BACKUP - LISTE DES PLANS
# =========================

# Liste tous les plans de backup existants
aws backup list-backup-plans

# Détaille un plan de backup spécifique
aws backup get-backup-plan \
  --backup-plan-id bp-12345678


# =========================
# AWS BACKUP - CREATION DE VAULT
# =========================

# Crée un coffre de sauvegarde (backup vault)
# --backup-vault-name : nom du coffre
aws backup create-backup-vault \
  --backup-vault-name MyVault

# Liste les coffres existants
aws backup list-backup-vaults


# =========================
# AWS BACKUP - START BACKUP JOB
# =========================

# Démarre une tâche de backup pour une ressource spécifique
# --backup-vault-name : coffre où stocker la sauvegarde
# --resource-arn : ARN de la ressource à sauvegarder (ex: volume EC2)
aws backup start-backup-job \
  --backup-vault-name MyVault \
  --resource-arn arn:aws:ec2:ap-south-1:123456789012:volume/vol-12345 \
  --iam-role-arn arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole

# Vérifie le statut d’un backup job
aws backup describe-backup-job \
  --backup-job-id 1234abcd-12ab-34cd-56ef-1234567890ab


# =========================
# AWS BACKUP - GESTION
# =========================

# Liste toutes les sauvegardes dans un coffre
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name MyVault

# Supprime un backup vault (doit être vide)
aws backup delete-backup-vault \
  --backup-vault-name MyVault

# Supprime un backup job (si nécessaire)
aws backup delete-recovery-point \
  --backup-vault-name MyVault \
  --recovery-point-arn arn:aws:backup:ap-south-1:123456789012:recovery-point:abcd1234
```

## Config

```Bash
# =========================
# AWS CONFIG - LISTE DES RÈGLES
# =========================

# Liste toutes les Config Rules existantes
aws config describe-config-rules

# Détaille une Config Rule spécifique
aws config describe-config-rules \
  --config-rule-names restricted-ssh


# =========================
# AWS CONFIG - COMPLIANCE
# =========================

# Vérifie la conformité des ressources selon une Config Rule
# --config-rule-name : nom de la règle (ex: restricted-ssh)
aws config get-compliance-details-by-config-rule \
  --config-rule-name restricted-ssh

# Filtre la conformité selon l’état : COMPLIANT / NON_COMPLIANT
aws config get-compliance-details-by-config-rule \
  --config-rule-name restricted-ssh \
  --compliance-types NON_COMPLIANT


# =========================
# AWS CONFIG - CRÉATION DE CONFIG RULE
# =========================

# Crée une règle gérée AWS (AWS Managed Rule) pour restreindre l’accès SSH public
aws config put-config-rule \
  --config-rule '{"ConfigRuleName":"restricted-ssh","Description":"Blocage accès SSH public","Source":{"Owner":"AWS","SourceIdentifier":"INCOMING_SSH_DISABLED"},"Scope":{"ComplianceResourceTypes":["AWS::EC2::SecurityGroup"]}}'

# Crée une règle personnalisée (Custom Lambda)
aws config put-config-rule \
  --config-rule '{
      "ConfigRuleName": "my-custom-rule",
      "Description": "Vérifie un paramètre spécifique",
      "Source": {
          "Owner": "CUSTOM_LAMBDA",
          "SourceIdentifier": "arn:aws:lambda:ap-south-1:123456789012:function:MyConfigLambda"
      }
  }'


# =========================
# AWS CONFIG - GESTION
# =========================

# Supprime une Config Rule
aws config delete-config-rule \
  --config-rule-name restricted-ssh

# Liste la conformité globale de toutes les règles
aws config get-compliance-summary-by-config-rule
```

## GuardDuty / Inspector

```Bash
# =========================
# AWS GUARDDUTY - LISTE DES DETECTEURS
# =========================

# Liste tous les détecteurs GuardDuty dans le compte
aws guardduty list-detectors

# Détaille un détecteur spécifique
aws guardduty get-detector \
  --detector-id abc123def456

# Liste les résultats (findings) pour un détecteur
aws guardduty list-findings \
  --detector-id abc123def456 \
  --finding-criteria '{"Criterion":{"severity":{"Gte":4}}}'  # exemple : sévérité >= 4

# Détaille un finding spécifique
aws guardduty get-findings \
  --detector-id abc123def456 \
  --finding-ids 12345678-1234-1234-1234-123456789012


# =========================
# AWS INSPECTOR2 - LISTE DES FINDINGS
# =========================

# Liste tous les findings d’Inspector2
aws inspector2 list-findings

# Filtrer les findings par sévérité ou état
aws inspector2 list-findings \
  --filter '{"severities":["HIGH","CRITICAL"],"status":["ACTIVE"]}'

# Détaille un finding spécifique
aws inspector2 describe-findings \
  --finding-arns arn:aws:inspector2:ap-south-1:123456789012:finding/abcd1234-5678-efgh-9012-ijkl34567890


# =========================
# AWS GUARDDUTY - CREATION DÉTECTEUR
# =========================

# Crée un détecteur GuardDuty (nécessaire pour activer la détection)
aws guardduty create-detector \
  --enable

# Active la protection multi-account (si organisation AWS)
aws guardduty enable-organization-admin-account \
  --admin-account-id 123456789012


# =========================
# AWS INSPECTOR2 - ACTIVATION
# =========================

# Active Inspector2 sur un compte ou une région
aws inspector2 enable \
  --resource-types "AWS_EC2","AWS_ECR"

# Active Inspector2 pour tous les comptes dans l’organisation
aws inspector2 enable-organization-admin-account \
  --admin-account-id 123456789012
```

## Cost Explorer / Billing

```Bash
# =========================
# AWS COST EXPLORER - REQUÊTE DES COÛTS
# =========================

# Récupère les coûts pour une période donnée
# --time-period : intervalle de dates (format YYYY-MM-DD)
# --granularity : DAILY, MONTHLY, HOURLY
# --metrics : type de coût (UnblendedCost, BlendedCost, AmortizedCost, NetAmortizedCost, NetUnblendedCost)
aws ce get-cost-and-usage \
  --time-period Start=2025-12-01,End=2025-12-04 \
  --granularity DAILY \
  --metrics "UnblendedCost"

# Exemple avec filtrage par service
aws ce get-cost-and-usage \
  --time-period Start=2025-12-01,End=2025-12-04 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --filter '{
    "Dimensions": {
      "Key": "SERVICE",
      "Values": ["Amazon EC2"]
    }
  }'

# Exemple avec regroupement par service
aws ce get-cost-and-usage \
  --time-period Start=2025-12-01,End=2025-12-04 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --group-by '[{"Type":"DIMENSION","Key":"SERVICE"}]'


# =========================
# AWS COST EXPLORER - GESTION
# =========================

# Liste les prévisions de coûts
aws ce get-forecast \
  --time-period Start=2025-12-05,End=2025-12-10 \
  --metric "UNBLENDED_COST" \
  --granularity DAILY \
  --filter '{
    "Dimensions": {
      "Key": "SERVICE",
      "Values": ["Amazon EC2"]
    }
  }'

# Liste les budgets définis (si AWS Budgets activé)
aws budgets describe-budgets \
  --account-id 123456789012
```

## CloudWatch Logs Insights

```Bash
# =========================
# CLOUDWATCH LOGS - QUERY INSIGHTS
# =========================

# Définir la période de la requête (les 10 dernières minutes)
START_TIME=$(date -d '10 minutes ago' +%s)
END_TIME=$(date +%s)

# Exécute une requête CloudWatch Logs Insights sur un groupe de logs Lambda
# --log-group-name : nom du groupe de logs
# --start-time / --end-time : intervalle en timestamp Unix
# --query-string : requête Insights (ex: récupérer timestamp + message, trier, limiter)
aws logs start-query \
  --log-group-name "/aws/lambda/myfunc" \
  --start-time $START_TIME \
  --end-time $END_TIME \
  --query-string "fields @timestamp, @message | sort @timestamp desc | limit 20"

# Obtenir l'état et les résultats de la requête
# Remplace QUERY_ID par l'identifiant retourné par start-query
aws logs get-query-results \
  --query-id QUERY_ID


# =========================
# CLOUDWATCH LOGS - LISTE DES GROUPES DE LOGS
# =========================

# Liste tous les groupes de logs
aws logs describe-log-groups

# Liste tous les streams d’un groupe spécifique
aws logs describe-log-streams \
  --log-group-name "/aws/lambda/myfunc" \
  --order-by "LastEventTime" \
  --descending


# =========================
# CLOUDWATCH LOGS - GESTION
# =========================

# Crée un groupe de logs
aws logs create-log-group \
  --log-group-name "/aws/lambda/myfunc"

# Crée un stream de logs
aws logs create-log-stream \
  --log-group-name "/aws/lambda/myfunc" \
  --log-stream-name "mystream"

# Supprime un groupe de logs (attention : supprime tout le contenu)
aws logs delete-log-group \
  --log-group-name "/aws/lambda/myfunc"
```

## Event Notifications

```Bash
# =========================
# EVENTBRIDGE - LISTE DES TARGETS D'UNE REGLE
# =========================

# Liste toutes les cibles attachées à une règle spécifique
# --rule : nom de la règle (ex: DailyBackup)
aws events list-targets-by-rule \
  --rule DailyBackup

# Exemple : filtrer ou formater le résultat avec jq
aws events list-targets-by-rule \
  --rule DailyBackup \
  | jq '.Targets[] | {Id, Arn, RoleArn}'

# =========================
# EVENTBRIDGE - GESTION DES TARGETS
# =========================

# Ajoute une cible à une règle (ex: Lambda)
aws events put-targets \
  --rule DailyBackup \
  --targets "Id"="1","Arn"="arn:aws:lambda:ap-south-1:123456789012:function:MyFunction"

# Supprime une cible d’une règle
aws events remove-targets \
  --rule DailyBackup \
  --ids "1"

# Vérifie si une cible spécifique est attachée
aws events list-targets-by-rule \
  --rule DailyBackup \
  | grep "MyFunction"
```

## EFS - Elastic File System

```Bash
# =========================
# EFS - LISTE DES FILE SYSTEMS
# =========================

# Liste tous les systèmes de fichiers EFS dans la région
aws efs describe-file-systems

# Détaille un système de fichiers spécifique
aws efs describe-file-systems \
  --file-system-id fs-12345678


# =========================
# EFS - CRÉATION DE FILE SYSTEM
# =========================

# Crée un nouveau système de fichiers EFS
# --creation-token : token unique pour l’idéntité de la création (évite les duplications)
aws efs create-file-system \
  --creation-token myefs \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --tags Key=Name,Value=MyEFS

# Exemple avec chiffrement et VPC
aws efs create-file-system \
  --creation-token myefs \
  --performance-mode generalPurpose \
  --throughput-mode provisioned \
  --encrypted \
  --kms-key-id arn:aws:kms:ap-south-1:123456789012:key/abcd1234 \
  --tags Key=Name,Value=MyEFS


# =========================
# EFS - GESTION
# =========================

# Crée un mount target pour permettre aux instances EC2 d’accéder au file system
aws efs create-mount-target \
  --file-system-id fs-12345678 \
  --subnet-id subnet-12345678 \
  --security-groups sg-12345678

# Supprime un file system
aws efs delete-file-system \
  --file-system-id fs-12345678

# Liste les mount targets d’un file system
aws efs describe-mount-targets \
  --file-system-id fs-12345678
```
