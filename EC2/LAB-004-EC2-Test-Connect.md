# LAB : Lancer une EC2 instance de test et s'y connecter

## Objectifs du Lab

Ce laboratoire te permet de **déployer et gérer une instance EC2 de test** sur AWS, en utilisant AWS CLI et PowerShell.  
À la fin de ce lab, tu seras capable de :

- Créer une paire de clés pour l'accès SSH
- Créer et configurer un **groupe de sécurité** pour autoriser le trafic SSH
- Lancer une instance EC2 avec Amazon Linux 2023 et un tag personnalisé
- Surveiller l'état de l'instance jusqu'à ce qu'elle soit opérationnelle
- Récupérer et afficher les informations de l'instance (ID, IP publique, réseau, stockage, AMI, VPC, tags, etc.)
- Se connecter à l'instance via SSH
- Supprimer proprement l'instance, la clé et le groupe de sécurité après utilisation

---

## Prérequis

Avant de commencer ce lab, tu dois disposer de :

- Un **compte Amazon Web Services actif**
- **AWS CLI** installé et configuré (`aws configure`)
- **PowerShell** pour exécuter les commandes
- Droits **EC2 complets** (créer instance, clé, security group)
- **SSH** installé pour se connecter à l’instance

## Prérequis

- Compte Amazon Web Services actif
- AWS CLI installé et configuré (`aws configure`)
- Utiliser PowerShell
- Avoir les droits EC2 (créer instance, clé, security group)
- Avoir SSH installé pour se connecter

## Remarque

Ce lab doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

## Création de la paire de clés

```PowerSehell
# Définir le chemin de destination
$DossierDeClé = "$HOME\Downloads\test-instance-key-pair.pem"

# Créer et enregistrer directement dans Téléchargements
aws ec2 create-key-pair `
  --key-name test-instance-key-pair `
  --key-type ed25519 `
  --query "KeyMaterial" `
  --output text | Out-File -Encoding ascii $DossierDeClé

Write-Host "Ta clé a été 'téléchargée' ici : $DossierDeClé" -ForegroundColor Green
```

## Création du Groupe de Sécurité

```PowerShell
# CRÉATION DU GROUPE
$SG_ID = aws ec2 create-security-group `
   --group-name "lab-ec2-test-sg" `
   --description "SG pour instance de test" `
   --query "GroupId" `
   --output text

# VÉRIFICATION ET AFFICHAGE
if ($SG_ID) {
    Write-Host "Groupe de sécurité créé avec l'ID : $SG_ID" -ForegroundColor Cyan
} else {
    Write-Error "Échec de la création du groupe de sécurité."
}
```

## Autorisation du flux SSH (Port 22)

```PowerShell
# Autoriser SSH (Port 22) depuis n'importe où
aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID `
  --protocol tcp `
  --port 22 `
  --cidr 0.0.0.0/0

Write-Host "Le port 22 est maintenant ouvert sur le groupe $SG_ID" -ForegroundColor Green
```

## Déploiement de l'instance

```PowerShell
# ==============================================================================
# DÉPLOIEMENT AUTOMATISÉ D'UNE INSTANCE AWS EC2
# Configuration : Amazon Linux 2023 | t3.micro | Key: test-instance-key-pair
# ==============================================================================

# RÉCUPÉRATION DE L'IMAGE (AMI)
$AMI_ID = aws ssm get-parameters `
    --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 `
    --query "Parameters[0].Value" `
    --output text

if ($AMI_ID -eq "None" -or [string]::IsNullOrEmpty($AMI_ID)) {
    Write-Error "Impossible de récupérer l'ID de l'AMI. Vérifiez votre région AWS."
    return
}

Write-Host "AMI ID identifiée : $AMI_ID" -ForegroundColor Cyan

# RÉCUPÉRATION DU GROUPE DE SÉCURITÉ (Nom corrigé)
# Rappel : Nous avons renommé 'sg-ec2-test' en 'lab-ec2-test-sg' pour respecter les règles AWS.
$SG_ID = aws ec2 describe-security-groups `
    --group-names "lab-ec2-test-sg" `
    --query "SecurityGroups[0].GroupId" `
    --output text

if ([string]::IsNullOrEmpty($SG_ID) -or $SG_ID -eq "None") {
    Write-Error "Le groupe de sécurité 'lab-ec2-test-sg' est introuvable. Veuillez le créer d'abord."
    return
}

Write-Host "Security Group ID : $SG_ID" -ForegroundColor Cyan

# LANCEMENT DE L'INSTANCE AVEC CLÉ ET SG
Write-Host "Initialisation de l'instance 'My EC2 instance'..." -ForegroundColor Yellow

$INSTANCE_ID = aws ec2 run-instances `
    --image-id $AMI_ID `
    --count 1 `
    --instance-type t3.micro `
    --key-name "test-instance-key-pair" `
    --security-group-ids $SG_ID `
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value="My EC2 instance"}]' `
    --query "Instances[0].InstanceId" `
    --output text

if ($null -eq $INSTANCE_ID) {
    Write-Error "Échec du lancement de l'instance."
    return
}

# MONITORING DU STATUT
Write-Host "Attente du passage en état 'Running' pour l'ID : $INSTANCE_ID"
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# RÉCUPÉRATION DE L'IP PUBLIQUE
# On attend un court instant pour s'assurer que l'IP est bien propagée
$PUBLIC_IP = aws ec2 describe-instances `
    --instance-ids $INSTANCE_ID `
    --query "Reservations[0].Instances[0].PublicIpAddress" `
    --output text

# RAPPORT DE DÉPLOIEMENT ET CONNEXION
Write-Host "`n------------------------------------------------------------" -ForegroundColor Green
Write-Host "                  RAPPORT DE SUCCÈS EC2" -ForegroundColor Green
Write-Host "------------------------------------------------------------" -ForegroundColor Green
Write-Host "ID Instance : $INSTANCE_ID"
Write-Host "IP Publique : $PUBLIC_IP"
Write-Host "Utilisateur : ec2-user"
Write-Host "------------------------------------------------------------"
Write-Host "COMMANDE DE CONNEXION SSH :" -ForegroundColor Yellow
Write-Host "ssh -i `"`$HOME\Downloads\test-instance-key-pair.pem`" ec2-user@$PUBLIC_IP"
```

## Afficher : ID, Propriétaire, Réseau, Stockage, VPC, AMI, etc.

```PowerShell
# RÉCUPÉRATION DE L'ID (On cherche par tag Name)
$TAG_NAME = "My EC2 instance"  # <--- VÉRIFIE BIEN CE NOM (Précédemment tu utilisais "My EC2 instance")

Write-Host "Recherche de l'instance nommée : '$TAG_NAME'..." -ForegroundColor Cyan

$INSTANCE_ID = aws ec2 describe-instances `
    --filters "Name=tag:Name,Values=$TAG_NAME" "Name=instance-state-name,Values=running,pending,stopped" `
    --query "Reservations[0].Instances[0].InstanceId" `
    --output text

# SÉCURITÉ : SI NON TROUVÉ, ON LISTE LES INSTANCES DISPONIBLES
if ($INSTANCE_ID -eq "None" -or [string]::IsNullOrEmpty($INSTANCE_ID)) {
    Write-Host "Aucune instance trouvée avec le nom '$TAG_NAME'." -ForegroundColor Red
    Write-Host "Voici les instances actuellement disponibles sur votre compte :" -ForegroundColor Yellow
    aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId, Tags[?Key=='Name'].Value | [0], State.Name]" --output table
    return
}

Write-Host "Instance détectée : $INSTANCE_ID" -ForegroundColor Green

# RÉCUPÉRATION DES DONNÉES
$data = aws ec2 describe-instances --instance-ids $INSTANCE_ID --query "Reservations[0].Instances[0]" --output json | ConvertFrom-Json
$status = aws ec2 describe-instance-status --instance-ids $INSTANCE_ID --query "InstanceStatuses[0]" --output json | ConvertFrom-Json
$ami_name = aws ec2 describe-images --image-ids $data.ImageId --query "Images[0].Name" --output text

# AFFICHAGE DU RAPPORT
Clear-Host
Write-Host "============================================================" -ForegroundColor White
Write-Host "  RAPPORT D'AUDIT : $INSTANCE_ID" -ForegroundColor Green
Write-Host "============================================================" -ForegroundColor White

Write-Host "[ CALCUL & OS ]" -ForegroundColor Yellow
$reportCalcul = [PSCustomObject]@{
    "Nom (Tag)"      = ($data.Tags | Where-Object {$_.Key -eq 'Name'}).Value
    "État"           = $data.State.Name.ToUpper()
    "Santé Système"  = if ($status.SystemStatus.Status) { $status.SystemStatus.Status } else { "Opérationnel (OK)" }
    "Type"           = $data.InstanceType
    "AMI ID"         = $data.ImageId
    "Nom de l'AMI"   = $ami_name
}
$reportCalcul | Format-List

Write-Host "[ RÉSEAU & VPC ]" -ForegroundColor Yellow
$reportReseau = [PSCustomObject]@{
    "VPC ID"         = $data.VpcId
    "IP Publique"    = if ($data.PublicIpAddress) { $data.PublicIpAddress } else { "Aucune" }
    "IP Privée"      = $data.PrivateIpAddress
    "DNS Public"     = if ($data.PublicDnsName) { $data.PublicDnsName } else { "N/A" }
}
$reportReseau | Format-List

Write-Host "[ SÉCURITÉ & ACCÈS ]" -ForegroundColor Yellow
$groupsText = ($data.SecurityGroups | ForEach-Object { "$($_.GroupName) ($($_.GroupId))" }) -join ", "
$reportSecu = [PSCustomObject]@{
    "Paire de Clés"  = $data.KeyName
    "Groupes Sec."   = $groupsText
}
$reportSecu | Format-List

Write-Host "[ STOCKAGE EBS ]" -ForegroundColor Yellow
$data.BlockDeviceMappings | ForEach-Object {
    Write-Host "- Device: $($_.DeviceName) | Volume: $($_.Ebs.VolumeId) | Statut: $($_.Ebs.Status)"
}

Write-Host "`n[ TAGS ]" -ForegroundColor Yellow
$data.Tags | Format-Table Key, Value -AutoSize

Write-Host "============================================================"
$keyPath = "$HOME\Downloads\$($data.KeyName).pem"
Write-Host "COMMANDE SSH :" -ForegroundColor Cyan
Write-Host "ssh -i `"$keyPath`" ec2-user@$($data.PublicIpAddress)"
```

## Se connecter à l'instance

```PowerShell
ssh -i "$HOME\Downloads\test-instance-key-pair.pem" ec2-user@35.181.151.40
```

## Résiliation

```PowerShell
# ==============================================================================
# SCRIPT DE NETTOYAGE COMPLET (CORRIGÉ)
# ==============================================================================

Write-Host "Début du nettoyage complet..." -ForegroundColor Red

# RÉCUPÉRATION ET SUPPRESSION DE L'INSTANCE
# On cherche les DEUX noms possibles pour être sûr de ne rien oublier
$TAG_NAMES = "My EC2 instance","EC2-PRO-LAB"

$INSTANCE_ID = aws ec2 describe-instances `
    --filters "Name=tag:Name,Values=$TAG_NAMES" "Name=instance-state-name,Values=running,stopped,pending" `
    --query "Reservations[0].Instances[0].InstanceId" `
    --output text

if ($INSTANCE_ID -ne "None" -and $INSTANCE_ID -ne $null) {
    Write-Host "Suppression de l'instance détectée : $INSTANCE_ID" -ForegroundColor Yellow
    aws ec2 terminate-instances --instance-ids $INSTANCE_ID | Out-Null

    Write-Host "Attente de la destruction complète (état 'terminated')..."
    aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
    Write-Host "Instance supprimée." -ForegroundColor Green
} else {
    Write-Host "Aucune instance active trouvée avec ces noms." -ForegroundColor Gray
}

# SUPPRESSION DU GROUPE DE SÉCURITÉ
# On utilise le nom correct que nous avons créé : "lab-ec2-test-sg"
$SG_NAME = "lab-ec2-test-sg"
$SG_ID = aws ec2 describe-security-groups `
    --group-names $SG_NAME `
    --query "SecurityGroups[0].GroupId" `
    --output text 2>$null

if ($SG_ID -ne "None" -and $SG_ID -ne $null) {
    Write-Host "Suppression du Groupe de Sécurité : $SG_NAME ($SG_ID)" -ForegroundColor Yellow
    # Petite pause pour laisser le réseau AWS se détacher complètement
    Start-Sleep -Seconds 2
    aws ec2 delete-security-group --group-id $SG_ID
    Write-Host "Groupe de sécurité supprimé." -ForegroundColor Green
}

# SUPPRESSION DE LA PAIRE DE CLÉS (AWS)
Write-Host "Suppression de la clé 'test-instance-key-pair' sur AWS..." -ForegroundColor Yellow
aws ec2 delete-key-pair --key-name "test-instance-key-pair"
Write-Host "Clé supprimée sur AWS." -ForegroundColor Green

# SUPPRESSION DU FICHIER LOCAL (.pem)
$PEM_PATH = "$HOME\Downloads\test-instance-key-pair.pem"
if (Test-Path $PEM_PATH) {
    Remove-Item $PEM_PATH -Force
    Write-Host "Fichier .pem local supprimé dans Téléchargements." -ForegroundColor Green
}

Write-Host "`Nettoyage terminé. Votre compte AWS est à nouveau vide !" -ForegroundColor Cyan
```
