# LAB : AWS CloudWatch – Monitoring, Alarms et Logs

Ce laboratoire montre comment :

- lister les **métriques CloudWatch**
- créer des **alarms** pour surveiller la CPU d’une instance **EC2**
- installer et configurer l’**agent CloudWatch** sur EC2
- collecter et visualiser des **logs système**

**CloudWatch** est un service AWS de **monitoring et observabilité** pour tes ressources et applications.

---

## Objectifs

- Surveiller les **métriques EC2** via CloudWatch
- Créer des **alarms** pour recevoir des notifications
- Configurer un **agent** pour collecter des logs système
- Explorer les **log groups** et **streams**

---

## Concepts clés

| Concept          | Description                                                         |
| ---------------- | ------------------------------------------------------------------- |
| Metric           | Une mesure numérique, par exemple CPUUtilization, NetworkIn         |
| Alarm            | Une règle qui déclenche une action si une métrique dépasse un seuil |
| Log Group        | Conteneur logique pour des logs similaires                          |
| Log Stream       | Suite chronologique de logs dans un log group                       |
| CloudWatch Agent | Agent installé sur EC2 pour collecter métriques et logs détaillés   |

## Remarque

Ce lab doit être réalisé dans **AWS CloudShell**.

## Déploiement

### Étape 1 : Lister les métriques disponibles

```Bash
aws cloudwatch list-metrics --region eu-west-3
```

![image](https://hackmd.io/_uploads/H11UsLb5bg.png)

![image](https://hackmd.io/_uploads/H1AFj8WcZl.png)

Permet de vérifier quelles métriques sont disponibles pour les instances EC2 et autres services.

### Création de la clé

```Bash
aws ec2 create-key-pair --key-name DevOpsKey --query 'KeyMaterial' --output text > DevOpsKey.pem
```

![image](https://hackmd.io/_uploads/rkVkh8Zc-x.png)

![image](https://hackmd.io/_uploads/HkzV-n8Z5Wl.png)

### Restreindre les permissions du fichier (Sécurité)

```Bash
chmod 400 DevOpsKey.pem
```

![image](https://hackmd.io/_uploads/S12zhLZcZl.png)

### Trouver votre VPC ID

```Bash
aws ec2 describe-vpcs --query "Vpcs[*].{ID:VpcId, Default:IsDefault}" --output table
```

![image](https://hackmd.io/_uploads/HyIE38Wq-e.png)

### Création du groupe

```Bash
aws ec2 create-security-group \
--group-name DevOpsSG \
--description "SG pour TP EC2" \
--vpc-id vpc-099163b43e6f5403d
```

> Pour le vpc-id : remplacer le par votre propre ID

![image](https://hackmd.io/_uploads/rkkwhU-9-g.png)

### Récupération de l'id du groupe

```Bash
aws ec2 describe-security-groups \
--filters "Name=group-name,Values=DevOpsSG" \
--query "SecurityGroups[*].GroupId" \
--output text
```

![image](https://hackmd.io/_uploads/HJy9hUb9-l.png)

### Réglage des ports (22 SSH) et permissions

```Bash
# On autorise le port 22 pour SSH
aws ec2 authorize-security-group-ingress --group-id sg-0a7b6c5d05e7fcf8e --protocol tcp --port 22 --cidr 0.0.0.0/0
```

> Pour le group-id : remplacer le par votre propre ID-Group

![image](https://hackmd.io/_uploads/B1qpn8Z9bx.png)

![image](https://hackmd.io/_uploads/SJkgaLZcWl.png)

### Récupération des infos (VPC-ID, Group-ID, Key-name, Subnet-ID)

```Bash
echo "--- RÉCAPITULATIF DE VOS RESSOURCES ---" && \
echo "VPC ID (Défaut) : " $(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text) && \
echo "Security Group ID : " $(aws ec2 describe-security-groups --filters "Name=group-name,Values=DevOpsSG" --query "SecurityGroups[0].GroupId" --output text) && \
echo "Key Pair Name : " $(aws ec2 describe-key-pairs --filters "Name=key-name,Values=DevOpsKey" --query "KeyPairs[0].KeyName" --output text) && \
echo "Subnet ID : " $(aws ec2 describe-subnets --filters "Name=default-for-az,Values=true" --query "Subnets[0].SubnetId" --output text)
```

![image](https://hackmd.io/_uploads/Sy1GaIb5-e.png)

### Lancement du EC2

> Rempalcer par vos vrais iidentifiants : Key-name, Group-ID, Subnet-ID

> L'image (AMI) de l'instance choisi est l'image par défaut

On passe à la création de l'instance

```Bash
aws ec2 run-instances \
--image-id ami-05d43d5e94bb6eb95 \
--count 1 \
--instance-type t2.micro \
--key-name DevOpsKey \
--security-group-ids sg-0a7b6c5d05e7fcf8e \
--subnet-id subnet-0a43e7ed25d656dad \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MonServeurDevOps}]'
```

![image](https://hackmd.io/_uploads/BydcTUWcbl.png)

![image](https://hackmd.io/_uploads/SkX2TI-qWx.png)

### Récupérer l’ID de l’instance

```Bash
INSTANCE_ID=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=MonServeurDevOps" \
--query "Reservations[0].Instances[0].InstanceId" \
--output text \
--region eu-west-3)

echo "Instance ID : $INSTANCE_ID"
```

![image](https://hackmd.io/_uploads/BJRp68W5Wl.png)

### Créer un topic SNS et subscription email

#### Créer le topic

```bash
aws sns create-topic --name NotifyMe --region eu-west-3
```

![image](https://hackmd.io/_uploads/rkheCL-5bx.png)

Note l’ARN retourné : `arn:aws:sns:eu-west-3:157125373775:NotifyMe`

![image](https://hackmd.io/_uploads/B1nBAIbcZx.png)

##### S’abonner par email

```Bash
aws sns subscribe \
--topic-arn arn:aws:sns:eu-west-3:157125373775:NotifyMe \
--protocol email \
--notification-endpoint pape.lo@estiam.com \
--region eu-west-3
```

![image](https://hackmd.io/_uploads/Hy5FCLW9Wx.png)

Confirmer l’abonnement depuis l’email reçu.

![image](https://hackmd.io/_uploads/S1ooA8ZcZx.png)

![image](https://hackmd.io/_uploads/rytACU-5Wl.png)

![image](https://hackmd.io/_uploads/rJIgkwb5be.png)

### Étape 3 : Créer l’alarme CPU CloudWatch

```Bash
aws cloudwatch put-metric-alarm \
--alarm-name HighCPU \
--metric-name CPUUtilization \
--namespace AWS/EC2 \
--statistic Average \
--period 300 \
--threshold 80 \
--comparison-operator GreaterThanThreshold \
--evaluation-periods 2 \
--alarm-actions arn:aws:sns:eu-west-3:157125373775:NotifyMe \
--dimensions Name=InstanceId,Value=i-040e7ea48656f1777 \
--region eu-west-3
```

Remplace `<ID-INSTANCE-EC2>` par l’ID réel de ton instance EC2.

![image](https://hackmd.io/_uploads/BJnFkPWcWl.png)

![image](https://hackmd.io/_uploads/H1V2kw-qZx.png)

![image](https://hackmd.io/_uploads/SJU6kP-5-e.png)

![image](https://hackmd.io/_uploads/rkPAywb9-g.png)

### Installer et configurer CloudWatch Agent

#### Se connecter à l’instance

```Bash
ssh -i DevOpsKey.pem ec2-user@<Public-IP>
```

![image](https://hackmd.io/_uploads/SyN4eDWq-e.png)

#### Installer l’agent

```bash
sudo yum install amazon-cloudwatch-agent -y
```

![image](https://hackmd.io/_uploads/HkhvePW5bx.png)

Vérifie que le paquet est installé avec

```Bash
rpm -qa | grep amazon-cloudwatch-agent
```

## Configurer l’agent via wizard

Avant de lancer le **wizard**, assure-toi que ton **EC2** a un **rôle IAM** attaché avec les permissions suivantes :

- **CloudWatchAgentServerPolicy** (AWS géré)
- **CloudWatchLogsFullAccess** (pour les log groups)
- **AmazonEC2ReadOnlyAccess** (optionnel, pour les dimensions EC2)

Ensuite, lance

#### Configurer via wizard

```Bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

![image](https://hackmd.io/_uploads/rJnaZwZcbe.png)

#### Attention aux points suivants dans le wizard

- **OS** : linux
- **Host type** : EC2
- **User** : cwagent (par défaut)
- **StatsD & CollectD** : no (si tu ne les utilises pas)
- **Host metrics** : yes
- **CPU metrics per core** : yes
- **EC2 dimensions** : yes
- **High-resolution metrics** : 60s (par défaut)
- **Default metrics config** : Basic
- **Log files** : yes
- **Log file path** : /var/log/messages
- **Log group name** : /devops-lab-logs
- **Log group class** : STANDARD
- **Log stream name** : {instance_id}
- **Retention** : 7 jours
- **Autres log files** : no
- **Store config in SSM** : no

Le wizard va générer :  
`/opt/aws/amazon-cloudwatch-agent/bin/config.json`

#### Démarrer et activer l’agent

Pour que l’agent utilise ton nouveau rôle IAM et la config

```Bash
# Arrêter l'agent s'il tourne encore
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop

# Appliquer la config et démarrer l'agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s

# Vérifier le statut
sudo systemctl status amazon-cloudwatch-agent
```

#### Le service doit être active (running).

### Générer un log

```Bash
echo "Test CloudWatch Agent $(date)" | sudo tee -a /var/log/messages
```

### Explorer les logs

#### Lister les log groups

```Bash
aws logs describe-log-groups --region eu-west-3
```

Tu devrais voir `/devops-lab-logs`.

#### Lister les log streams d’un groupe

```Bash
aws logs describe-log-streams --log-group-name /devops-lab-logs --region eu-west-3
```

Le log stream `{instance_id}` devrait apparaître et contenir ton test log.

### Nettoyage

#### Terminer l'instance (Destruction)

```Bash
# On récupère l'ID de l'instance via son nom pour la supprimer
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=MonServeurDevOps" "Name=instance-state-name,Values=running,pending,stopped" --query "Reservations[0].Instances[0].InstanceId" --output text)

aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

#### Supprimer le Security Group

```Bash
# 1. On récupère l'ID via le nom "DevOpsSG"
SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=DevOpsSG" --query "SecurityGroups[0].GroupId" --output text)

# 2. On le supprime
aws ec2 delete-security-group --group-id $SG_ID
```

**Pourquoi ça pourrait échouer ?**

Si vous recevez un message d'erreur de type `DependencyViolation`, c'est pour l'une de ces deux raisons :

- **L'instance est encore "en vie"** : Elle doit être à l'état `terminated` (totalement disparue de la console) avant que le groupe soit libéré.

- **Règles croisées** : Un autre groupe de sécurité possède une règle qui fait référence à celui-ci.

#### Supprimer la Key Pair

```Bash
# Supprime la clé du catalogue AWS
aws ec2 delete-key-pair --key-name DevOpsKey

# Supprime le fichier sur votre ordinateur
rm DevOpsKey.pem
```

#### CloudWatch

```Bash
# Supprimer l'alarme
aws cloudwatch delete-alarms --alarm-names HighCPU --region eu-west-3

# Supprimer le topic SNS
aws sns delete-topic --topic-arn arn:aws:sns:eu-west-3:157125373775:NotifyMe --region eu-west-3

# Supprimer les logs CloudWatch (si créés)
aws logs delete-log-group --log-group-name /devops-lab-logs --region eu-west-3
```
