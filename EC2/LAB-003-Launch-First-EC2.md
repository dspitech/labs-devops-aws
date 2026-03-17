# LAB — Lancer ta première instance EC2 (CloudShell)

## Objectif du lab

L’objectif de ce lab est de découvrir comment lancer et gérer une instance EC2 avec AWS CLI depuis CloudShell.  
À la fin, tu sauras :

- Déployer une instance EC2 (Amazon Linux 2023)
- Vérifier son état et récupérer ses informations
- Supprimer proprement les ressources

---

## Contexte

Ce lab simule un premier déploiement simple d’une machine virtuelle dans le cloud (EC2), sans configuration SSH ni clé, afin de se concentrer sur les bases du cloud AWS.

## NB :

## Ce lab doit être réalisé dans **AWS CloudShell** pas dans **PowerShell**.

## Prérequis

- Avoir un compte AWS actif
- Être connecté à AWS CloudShell
- Avoir les droits EC2 (lancer, décrire, supprimer une instance)
- Connaître les bases des commandes Linux (bash)

## Vérifier la version du cli

```Bash
aws --version
```

## Savoir qui tu es côté AWS (quelle identité utilise la CLI).

```Bash
aws sts get-caller-identity
```

## Récupérer l’AMI Amazon Linux 2023

Cette commande suivante récupère automatiquement l’ID de la dernière **Amazon Linux 2023 AMI** via AWS SSM, le stocke dans la variable `AMI_ID` et l’affiche avec `echo`.

```Bash
AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query "Parameters[0].Value" \
  --output text)

echo "AMI ID: $AMI_ID"
```

## Récupérer le Security Group par défaut

Cette commande suivante récupère l’**ID du Security Group par défaut** d’un VPC, le stocke dans la variable `SG_ID` et l’affiche avec `echo`.

```Bash
SG_ID=$(aws ec2 describe-security-groups \
  --group-names default \
  --query "SecurityGroups[0].GroupId" \
  --output text)

echo "Security Group ID: $SG_ID"
```

## Lancer l’instance EC2

Cette commande suivante **crée une instance EC2** avec l’AMI, le type et le groupe de sécurité spécifiés, stocke son **ID dans `INSTANCE_ID`** et l’affiche.

```Bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --count 1 \
  --instance-type t3.micro \
  --security-group-ids $SG_ID \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value="My EC2 instance"}]' \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Instance ID: $INSTANCE_ID"
```

## Attendre qu’elle soit en RUNNING

Cette commande suivante attend que l’instance EC2 identifiée par `$INSTANCE_ID` passe à l’état **running**, puis affiche un message de confirmation.

```Bash
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance en cours d'exécution"
```

## Afficher les infos principales

Cette commande suivante affiche les informations principales de l’instance EC2 identifiée par `$INSTANCE_ID` :

- Nom (tag `Name`)
- ID de l’instance
- Type d’instance
- IP publique
- État de l’instance

```Bash
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].[Tags[?Key=='Name'].Value | [0], InstanceId, InstanceType, PublicIpAddress, State.Name]" \
  --output table
```

## Lister toutes les instances

Cette commande liste **toutes les instances EC2 en cours d’exécution** dans le compte avec les informations suivantes :

- Nom (tag `Name`)
- ID de l’instance
- IP publique
- Date et heure de lancement

```Bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name'].Value | [0], ID:InstanceId, IP:PublicIpAddress, LaunchTime:LaunchTime}" \
  --output table
```

## Supprimer l’instance

Cette commande suivante **supprime (termine) l’instance EC2** identifiée par `$INSTANCE_ID`.

```Bash
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

Cette commande suivante attend que l’instance EC2 identifiée par `$INSTANCE_ID` soit **complètement supprimée**, puis affiche un message de confirmation.

```Bash
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
echo "Instance supprimée avec succès"
```

## Vérifier si l'instance EC2 est supprimée

```Bash
aws ec2 describe-instances --instance-ids $INSTANCE_ID --query "Reservations[0].Instances[0].State.Name" --output text
```
