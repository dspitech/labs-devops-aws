# LAB : Dépoiement d'un Site WordPress sur AWS avec MySQL

## Objectif du projet

- Héberger WordPress sur une machine virtuelle (EC2).
- Externaliser la base de données sur un service managé (RDS) pour plus de fiabilité.
- Sécuriser l'accès : La base de données est privée (invisible sur Internet), seul le serveur Web peut lui parler.
- Automatisation totale : En un clic, le serveur s'installe, télécharge WordPress et se connecte à la base sans intervention manuelle.

## Remarque

Ce lab doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

## Architecture

![image](https://hackmd.io/_uploads/Sy9YXdVKWe.png)

- **EC2** → Serveur Web Apache + PHP + WordPress
- **RDS MySQL** → Base de données managée
- **Security Groups** → Frontend Web et Backend RDS
- **User Data script** → Installation automatique de WordPress et configuration de `wp-config.php`

## Se connecter à la console AWS

![image](https://hackmd.io/_uploads/H1_PI_NKWx.png)

## Autorisation l'exécution de scripts & installation des dépendances

```PowerShell
Set-ExecutionPolicy RemoteSigned -Force

# Installation du module AWS (si pas déjà fait)
Install-Module -Name AWS.Tools.Installer -Force -AllowClobber

# Installation des composants spécifiques EC2 et RDS
Install-AWSToolsModule AWS.Tools.EC2, AWS.Tools.RDS -CleanUp
```

![image](https://hackmd.io/_uploads/S1cc8ONYbe.png)

## Création des dossiers et fichiers du projet

Lance ce bloc de code dans PowerShell. Il va créer le dossier sur ton bureau et générer les fichiers de confguration.

```PowerShell
# Définir le chemin du projet (sur le Bureau)
$Path = Join-Path $home "Desktop\Projet-WordPress-AWS"

# Créer le dossier racine
if (!(Test-Path $Path)) {
    New-Item -ItemType Directory -Path $Path
    Write-Host "[OK] Dossier créé : $Path" -ForegroundColor Green
}

# Créer les fichiers de script
$Files = @("01_Deploy_Infra.ps1", "02_Get_Credentials.ps1", "03_Cleanup.ps1")

foreach ($File in $Files) {
    $FilePath = Join-Path $Path $File
    if (!(Test-Path $FilePath)) {
        New-Item -ItemType File -Path $FilePath
        Write-Host "[+] Fichier créé : $File" -ForegroundColor Cyan
    }
}

# Créer un petit fichier mémo
$Memo = @"
PROJET WORDPRESS RDS - DSPI-TECH
--------------------------------
Identifiants par défaut :
DB Name : wordpress_db
Admin : wp_db_admin
Pass : P@ssw0rdSecure2026!
Region : eu-west-3
"@
Set-Content -Path (Join-Path $Path "README.txt") -Value $Memo

Write-Host "`nStructure de projet prête ! Tu peux maintenant copier les codes dedans." -ForegroundColor Yellow
```

![image](https://hackmd.io/_uploads/ryEdPu4Y-g.png)
![image](https://hackmd.io/_uploads/H1dcD_VFbx.png)
![image](https://hackmd.io/_uploads/B1ly2wuVYZl.png)
![image](https://hackmd.io/_uploads/HkzGOdEKbl.png)

## Déploiement de l'infrastructure

- 🖥️ EC2 → Serveur Web (Apache + PHP + WordPress)
- 🗄️ RDS MySQL → Base de données managée
- 🔐 Security Groups → Sécurisation réseau
- ⚙️ Configuration automatique de wp-config.php

Dans `01_Deploy_Infra.ps1` : Colle ce script suivant & exécute le :

```PowerShell
<#
.SYNOPSIS
    Projet : Déploiement WordPress (EC2 + RDS).
#>

# --- 1. PARAMÈTRES ET CONFIGURATION ---
$Config = @{
    Region          = "eu-west-3"
    ProjectName     = "WordPress-Pro-Lab"
    DBInstanceId    = "rds-wp-prod"
    DBName          = "wordpress_db"
    MasterUser      = "wp_db_admin"
    MasterPass      = "WPAdminSecure2026"
    WebSGName       = "web-wordpress-frontend"
    DBSGName        = "db-wordpress-backend"
    InstanceType    = "t3.micro" # Instance EC2 éligible Free Tier
}

$GlobalTimer = [System.Diagnostics.Stopwatch]::StartNew()

try {
    Write-Host "`n[1/5] ANALYSE DU RESEAU ET DU VPC" -ForegroundColor Cyan
    $VpcId = (aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text --region $Config.Region)

    if ($VpcId -eq "None" -or !$VpcId) {
        throw "VPC par defaut introuvable."
    }

    # --- SG WEB ---
    $WebSGId = (aws ec2 describe-security-groups --filters "Name=group-name,Values=$($Config.WebSGName)" --query "SecurityGroups[0].GroupId" --output text --region $Config.Region)
    if ($WebSGId -eq "None" -or !$WebSGId) {
        $WebSGId = (aws ec2 create-security-group --group-name $Config.WebSGName --description "Front-end WP" --vpc-id $VpcId --query "GroupId" --output text --region $Config.Region)
        aws ec2 authorize-security-group-ingress --group-id $WebSGId --protocol tcp --port 80 --cidr 0.0.0.0/0 --region $Config.Region
        aws ec2 authorize-security-group-ingress --group-id $WebSGId --protocol tcp --port 22 --cidr 0.0.0.0/0 --region $Config.Region
    }

    # --- SG RDS ---
    $DBSGId = (aws ec2 describe-security-groups --filters "Name=group-name,Values=$($Config.DBSGName)" --query "SecurityGroups[0].GroupId" --output text --region $Config.Region)
    if ($DBSGId -eq "None" -or !$DBSGId) {
        $DBSGId = (aws ec2 create-security-group --group-name $Config.DBSGName --description "Back-end RDS" --vpc-id $VpcId --query "GroupId" --output text --region $Config.Region)
        aws ec2 authorize-security-group-ingress --group-id $DBSGId --protocol tcp --port 3306 --source-group $WebSGId --region $Config.Region
    }

    Write-Host "`n[2/5] PROVISIONNEMENT RDS (Optimisation Free Tier)" -ForegroundColor Cyan

    $CheckRDS = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --region $Config.Region 2>$null)

    if (!$CheckRDS) {
        Write-Host "[-] Lancement de la base RDS avec options d'economie..." -ForegroundColor Yellow
        # AJOUT DES PARAMÈTRES D'OPTIMISATION ICI
        aws rds create-db-instance `
            --db-instance-identifier $Config.DBInstanceId `
            --db-name $Config.DBName `
            --engine mysql `
            --db-instance-class db.t3.micro `
            --allocated-storage 20 `
            --master-username $Config.MasterUser `
            --master-user-password $Config.MasterPass `
            --vpc-security-group-ids $DBSGId `
            --no-multi-az `
            --backup-retention-period 0 `
            --monitoring-interval 0 `
            --no-enable-performance-insights `
            --auto-minor-version-upgrade `
            --no-publicly-accessible `
            --region $Config.Region | Out-Null

        Write-Host "[OK] Commande de creation envoyee." -ForegroundColor Green
    }

    Write-Host "[-] Attente que la base RDS soit disponible..." -NoNewline
    do {
        Write-Host "." -NoNewline
        Start-Sleep -Seconds 30
        $Status = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --query "DBInstances[0].DBInstanceStatus" --output text --region $Config.Region 2>$null)
        if (!$Status) { $Status = "creating" }
    } while ($Status -ne "available")

    $RDS_Host = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --query "DBInstances[0].Endpoint.Address" --output text --region $Config.Region)
    Write-Host "`n[OK] RDS operationnel : $RDS_Host" -ForegroundColor Green

    Write-Host "`n[3/5] LANCEMENT EC2" -ForegroundColor Cyan
    $AMI = (aws ec2 describe-images --owners amazon --filters "Name=name,Values=al2023-ami-2023*kernel-6.1-x86_64" --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text --region $Config.Region)

    $UserScript = @"
#!/bin/bash
dnf update -y
dnf install httpd php8.2 php8.2-mysqlnd mariadb105 -y
cd /var/www/html
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz --strip-components=1
cp wp-config-sample.php wp-config.php
sed -i "s/database_name_here/$($Config.DBName)/" wp-config.php
sed -i "s/username_here/$($Config.MasterUser)/" wp-config.php
sed -i "s/password_here/$($Config.MasterPass)/" wp-config.php
sed -i "s/localhost/$RDS_Host/" wp-config.php
chown -R apache:apache /var/www/html
systemctl enable --now httpd
"@
    $UserData64 = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($UserScript))

    $InstanceId = (aws ec2 run-instances --image-id $AMI --count 1 --instance-type $Config.InstanceType --security-group-ids $WebSGId --user-data $UserData64 --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$($Config.ProjectName)}]" --query "Instances[0].InstanceId" --output text --region $Config.Region)

    Start-Sleep -Seconds 15
    $PublicIP = (aws ec2 describe-instances --instance-ids $InstanceId --query "Reservations[0].Instances[0].PublicIpAddress" --output text --region $Config.Region)

    Write-Host "`n[4/5] FINALISATION" -ForegroundColor Green
    Write-Host "--------------------------------------------------------"
    Write-Host " URL DU SITE    : http://$PublicIP" -ForegroundColor Cyan
    Write-Host " RDS ENDPOINT   : $RDS_Host"
    Write-Host " TEMPS TOTAL    : $([math]::Round($GlobalTimer.Elapsed.TotalMinutes, 2)) min"
    Write-Host "--------------------------------------------------------"

} catch {
    Write-Host "`n[ERREUR] : $($_.Exception.Message)" -ForegroundColor Red
}
```

![image](https://hackmd.io/_uploads/HyAmlYEt-e.png)
![image](https://hackmd.io/_uploads/SybHetNtbl.png)
![image](https://hackmd.io/_uploads/SJTYVF4K-l.png)

- Groupe de sécurité
  ![image](https://hackmd.io/_uploads/Hy4CVY4FWx.png)
  ![image](https://hackmd.io/_uploads/rJExSKEF-e.png)
- RDS
  ![image](https://hackmd.io/_uploads/BkcNrYVF-g.png)
  ![image](https://hackmd.io/_uploads/rJzISFEtWe.png)
- EC2
  ![image](https://hackmd.io/_uploads/ry7YBFVYWg.png)

## Ce que fait réellement le script

- Crée les Security Groups
- Déploie une base MySQL RDS privée
- Attend sa disponibilité
- Déploie une EC2
- Installe Apache + PHP
- Télécharge WordPress
- Configure automatiquement wp-config
- Lance le serveur
- Affiche l’URL

## Extraction des identifiants

Ce script PowerShell permet de récupérer automatiquement les informations de connexion d’un projet WordPress déployé sur AWS :

- Base de données RDS
- Serveur Web EC2
- URL publique du site

Dans `02_Get_Credentials.ps1` : Colle ce script suivant & exécute le :

```PowerShell
<#
.SYNOPSIS
    Recupere les informations de connexion du projet WordPress (EC2 + RDS).
#>

# --- CONFIGURATION ---
$Config = @{
    DBInstanceId = "rds-wp-prod"
    ProjectTag   = "WordPress-Pro-Lab"
    Region       = "eu-west-3"
    MasterPass   = "WPAdminSecure2026"
}

try {
    Write-Host "`n--- RECUPERATION DES PARAMETRES DE CONNEXION ---" -ForegroundColor Cyan

    # 1. Recuperation des infos RDS via AWS CLI
    $RDSInfo = aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --region $Config.Region --output json | ConvertFrom-Json
    $Endpoint = $RDSInfo.DBInstances[0].Endpoint.Address
    $DBName   = $RDSInfo.DBInstances[0].DBName
    $DBUser   = $RDSInfo.DBInstances[0].MasterUsername
    $DBStatus = $RDSInfo.DBInstances[0].DBInstanceStatus

    # 2. Recuperation de l'IP Publique EC2
    $PublicIP = (aws ec2 describe-instances --filters "Name=tag:Name,Values=$($Config.ProjectTag)" "Name=instance-state-name,Values=running" --query "Reservations[0].Instances[0].PublicIpAddress" --output text --region $Config.Region)

    # 3. AFFICHAGE
    Write-Host "`n========================================================" -ForegroundColor Gray
    Write-Host "  INFOS BASE DE DONNEES (RDS) - Statut: $DBStatus" -ForegroundColor Yellow
    Write-Host "--------------------------------------------------------"
    Write-Host " ENDPOINT     : " -NoNewline; Write-Host $Endpoint -ForegroundColor Cyan
    Write-Host " NOM DB       : " -NoNewline; Write-Host $DBName -ForegroundColor White
    Write-Host " ADMIN USER   : " -NoNewline; Write-Host $DBUser -ForegroundColor White
    Write-Host " MOT DE PASSE : " -NoNewline; Write-Host $Config.MasterPass -ForegroundColor White

    Write-Host "`n  INFOS SERVEUR WEB (EC2)" -ForegroundColor Yellow
    Write-Host "--------------------------------------------------------"
    if ($PublicIP -eq "None" -or [string]::IsNullOrEmpty($PublicIP)) {
        Write-Host " IP PUBLIQUE  : " -NoNewline; Write-Host "Non disponible ou instance arretee" -ForegroundColor Red
    } else {
        Write-Host " IP PUBLIQUE  : " -NoNewline; Write-Host $PublicIP -ForegroundColor Cyan
        Write-Host " URL DU SITE  : " -NoNewline; Write-Host "http://$PublicIP" -ForegroundColor Green
    }
    Write-Host "========================================================" -ForegroundColor Gray

} catch {
    Write-Host "[ERREUR] Impossible de recuperer les donnees. Verifiez les ressources." -ForegroundColor Red
}
```

![image](https://hackmd.io/_uploads/HySuItVFZx.png)

## Se connecter au site web

![image](https://hackmd.io/_uploads/rkDjUY4YWx.png)

## Nettoyage

Ce script PowerShell supprime toutes les ressources AWS créées pour le projet WordPress afin d’éviter tout coût supplémentaire.

### 🎯 Objectif :

Garantir qu’aucune ressource payante ne reste active dans le compte AWS.

- Ressources Concernées
- Instance EC2
- Instance RDS
- Security Groups (Frontend + Backend)

> Région utilisée : eu-west-3 (Paris)

Dans 03_Cleanup.ps1 : Colle le script pour tout supprimer et exécute le.

```PowerShell
<#
.SYNOPSIS
    Nettoyage complet du projet WordPress (EC2 + RDS + Security Groups).

#>

$Config = @{
    DBInstanceId = "rds-wp-prod"
    ProjectTag   = "WordPress-Pro-Lab"
    WebSGName    = "web-wordpress-frontend"
    DBSGName     = "db-wordpress-backend"
    Region       = "eu-west-3"
}

try {
    Write-Host "`n[!] DEMARRAGE DU NETTOYAGE GLOBAL..." -ForegroundColor Magenta
    Write-Host "--------------------------------------------------------"

    # 1. SUPPRESSION DE L'INSTANCE EC2
    Write-Host "[-] Recherche de l'instance EC2..." -ForegroundColor Yellow
    $InstanceId = (aws ec2 describe-instances --filters "Name=tag:Name,Values=$($Config.ProjectTag)" "Name=instance-state-name,Values=running,pending,stopped" --query "Reservations[0].Instances[0].InstanceId" --output text --region $Config.Region)

    if ($InstanceId -and $InstanceId -ne "None") {
        aws ec2 terminate-instances --instance-ids $InstanceId --region $Config.Region | Out-Null
        Write-Host "[OK] Instance $InstanceId en cours de resiliation." -ForegroundColor Green
    } else {
        Write-Host "[i] Aucune instance EC2 active trouvee." -ForegroundColor Gray
    }

    # 2. SUPPRESSION DE L'INSTANCE RDS
    Write-Host "[-] Suppression de l'instance RDS (Sans Snapshot)..." -ForegroundColor Yellow
    $RDSStatus = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --query "DBInstances[0].DBInstanceStatus" --output text --region $Config.Region 2>$null)

    if ($RDSStatus -and $RDSStatus -ne "None") {
        aws rds delete-db-instance `
            --db-instance-identifier $Config.DBInstanceId `
            --skip-final-snapshot `
            --delete-automated-backups `
            --region $Config.Region | Out-Null
        Write-Host "[OK] Suppression RDS lancee (Statut actuel: $RDSStatus)." -ForegroundColor Green
    } else {
        Write-Host "[i] Aucune instance RDS trouvee." -ForegroundColor Gray
    }

    # 3. ATTENTE DE LIBERATION DES RESSOURCES (Crucial pour les Security Groups)
    Write-Host "[-] Attente de la liberation totale des ressources..." -NoNewline
    do {
        Write-Host "." -NoNewline
        Start-Sleep -Seconds 30

        # Verif EC2
        $CheckEC2 = (aws ec2 describe-instances --filters "Name=tag:Name,Values=$($Config.ProjectTag)" "Name=instance-state-name,Values=shutting-down,running" --query "Reservations[0].Instances[0].InstanceId" --output text --region $Config.Region)

        # Verif RDS
        $CheckRDS = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --region $Config.Region 2>$null)

    } while (($CheckEC2 -and $CheckEC2 -ne "None") -or $CheckRDS)

    Write-Host "`n[OK] Ressources EC2/RDS liberees." -ForegroundColor Green

    # 4. SUPPRESSION DES SECURITY GROUPS
    Write-Host "[-] Nettoyage des Security Groups..." -ForegroundColor Yellow

    # Ordre de suppression inverse (RDS d'abord car il depend potentiellement du WebSG)
    $SGs = @($Config.DBSGName, $Config.WebSGName)
    foreach ($SGName in $SGs) {
        $SGId = (aws ec2 describe-security-groups --filters "Name=group-name,Values=$SGName" --query "SecurityGroups[0].GroupId" --output text --region $Config.Region)

        if ($SGId -and $SGId -ne "None") {
            aws ec2 delete-security-group --group-id $SGId --region $Config.Region
            Write-Host "[OK] Security Group $SGName ($SGId) supprime." -ForegroundColor Green
        }
    }

    Write-Host "`n[SUCCESS] NETTOYAGE TERMINE : Votre compte est propre." -ForegroundColor Cyan
    Write-Host "========================================================"

} catch {
    Write-Host "`n[ERREUR] Une ressource n'a pas pu etre supprimee : $($_.Exception.Message)" -ForegroundColor Red
}
```

![image](https://hackmd.io/_uploads/ry5hPtNKWl.png)
![image](https://hackmd.io/_uploads/BJp0DYEF-e.png)
![image](https://hackmd.io/_uploads/HJpgOYEt-g.png)
![image](https://hackmd.io/_uploads/SkwYOt4FWl.png)
