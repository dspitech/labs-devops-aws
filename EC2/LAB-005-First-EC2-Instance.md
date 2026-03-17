# LAB : Lancer ma toute première EC2 instance Amazon

## Objectifs du Lab

Ce laboratoire a pour objectif de te familiariser avec le **déploiement d’une instance EC2** sur AWS et la vérification de sa configuration via AWS CLI.  
À la fin de ce lab, tu seras capable de :

- Vérifier la configuration AWS CLI et l’identité de l’utilisateur
- Vérifier les groupes IAM et leurs permissions
- Lancer une instance EC2 avec un AMI spécifique et un type donné
- Surveiller l’état d’une instance jusqu’à ce qu’elle soit opérationnelle
- Afficher les informations principales d’une instance (ID, IP, tags, date de lancement, état)
- Supprimer proprement une instance EC2

---

## Prérequis

Avant de commencer ce lab, tu dois disposer de :

- Un **compte AWS** avec droits suffisants pour lancer des instances et accéder à IAM
- **AWS CLI** installé et configuré
- Connaissances de base en **ligne de commande / PowerShell**
- Accès à **PowerShell** ou tout terminal compatible pour exécuter les commandes

---

## Remarque

Ce lab doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

## Vérification de la configuration AWS

### Voir la version AWS CLI

```PowerShell
aws --version

```

### Voir qui est connecté

```PowerShell
aws sts get-caller-identity
```

### Voir le groupe de cet utilisateur

```PowerShell
aws iam list-groups-for-user --user-name DspiAdmin --query "Groups[*].GroupName" --output table
```

### Voir les autorisations du groupe

```PowerShell
aws iam list-attached-group-policies --group-name Administrators --query "AttachedPolicies[*].[PolicyName,PolicyArn]" --output table
```

## Déploiement de l'instance EC2

### Configuration

- AMI : Amazon Linux 2023
- Type d'instance : t3.micro
- Tag : My EC2 instance

```PowerShell
# Déploiement de l'instance

# ==============================================================================
# DÉPLOIEMENT AUTOMATISÉ D'UNE INSTANCE AWS EC2
# Configuration : Amazon Linux 2023 | t3.micro | Tag: My EC2 instance
# ==============================================================================

# RÉCUPÉRATION DE L'IMAGE (AMI)
# Utilisation du service SSM pour obtenir l'ID de la dernière version stable d'AL2023.
# Cette méthode évite les erreurs de filtrage manuel et est plus rapide.
$AMI_ID = aws ssm get-parameters `
    --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 `
    --query "Parameters[0].Value" `
    --output text

if ($AMI_ID -eq "None" -or [string]::IsNullOrEmpty($AMI_ID)) {
    Write-Error "Impossible de récupérer l'ID de l'AMI. Vérifiez votre région AWS."
    return
}

Write-Host "AMI ID identifiée : $AMI_ID" -ForegroundColor Cyan

# RÉCUPÉRATION DES PARAMÈTRES RÉSEAU
# Extraction de l'ID du groupe de sécurité par défaut du VPC.
$SG_ID = aws ec2 describe-security-groups `
    --group-names default `
    --query "SecurityGroups[0].GroupId" `
    --output text

# LANCEMENT DE L'INSTANCE
# Déploiement d'une instance de type t3.micro avec application immédiate du tag 'Name'.
Write-Host "Initialisation de l'instance 'My EC2 instance'..." -ForegroundColor Yellow

$INSTANCE_ID = aws ec2 run-instances `
    --image-id $AMI_ID `
    --count 1 `
    --instance-type t3.micro `
    --security-group-ids $SG_ID `
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value="My EC2 instance"}]' `
    --query "Instances[0].InstanceId" `
    --output text

if ($null -eq $INSTANCE_ID) {
    Write-Error "Échec du lancement de l'instance."
    return
}

# MONITORING DU STATUT
# Attente active jusqu'à ce que l'instance soit prête (état : running).
Write-Host "⏳ Attente du passage en état 'Running' pour l'ID : $INSTANCE_ID"
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# RAPPORT DE DÉPLOIEMENT
# Affichage final des informations de connexion et de l'état de l'instance.
Write-Host "`n------------------------------------------------------------" -ForegroundColor Green
Write-Host "                 RAPPORT DE SUCCÈS EC2" -ForegroundColor Green
Write-Host "------------------------------------------------------------" -ForegroundColor Green

aws ec2 describe-instances `
    --instance-ids $INSTANCE_ID `
    --query "Reservations[0].Instances[0].[Tags[?Key=='Name'].Value | [0], InstanceId, InstanceType, PublicIpAddress, State.Name]" `
    --output table
```

### Afficher les infos de l' instance (ID - IP publique - Propriétaire - Tags - Haure de lancement - Etat de l'instance)

```PowerShell
aws ec2 describe-instances `
    --filters "Name=instance-state-name,Values=running" `
    --query "Reservations[].Instances[].{Name:Tags[?Key=='Name'].Value | [0], ID:InstanceId, IP:PublicIpAddress, LaunchTime:LaunchTime}" `
    --output table
```

### Résiliation de l'instance

```PowerShell
Write-Host "Suppression définitive en cours..." -ForegroundColor Yellow
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
Write-Host "L'instance a été supprimée avec succès." -ForegroundColor Green
```
