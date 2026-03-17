# LAB : Créer et se connecter à une base de données MySQL avec Amazon RDS

## Qu’est-ce qu’Amazon RDS ?

Amazon RDS (Relational Database Service) est un service géré par AWS qui simplifie la configuration, l’exploitation et la mise à l’échelle d’une base de données relationnelle dans le cloud.
Avec RDS, AWS gère les tâches courantes comme la sauvegarde, la mise à jour logicielle, le monitoring et la scalabilité, afin que tu puisses te concentrer sur ton application plutôt que sur l’administration de la base de données.

## Avantages clés d’Amazon RDS

- **Service géré** : pas besoin d’installer, configurer ou patcher la base de données.
- **Scalabilité facile** : tu peux ajuster la capacité de calcul et de stockage selon tes besoins.
- **Haute disponibilité** : possibilité d’utiliser Multi-AZ pour assurer la continuité de service.
- **Sécurité intégrée** : chiffrement des données au repos et en transit, gestion IAM pour le contrôle d’accès.
- **Sauvegardes automatiques** : sauvegardes quotidiennes et snapshots manuels.
- **Monitoring et alertes** : intégration avec CloudWatch pour suivre les métriques de performance.

## Types de moteurs pris en charge

Amazon RDS supporte plusieurs moteurs de base de données relationnelle

- MySQL
- PostgreSQL
- MariaDB
- Oracle
- SQL Server
- Amazon Aurora

## Cas d’usage typiques

- Applications web nécessitant une base relationnelle.
- Applications critiques nécessitant haute disponibilité et sauvegardes automatiques.
- Projets nécessitant une scalabilité facile sans administration complexe.
- Développement et test rapide sans se soucier de la configuration du serveur de base de données.

## Configuration

### Étape 1 : Créer une instance de base de données MySQL

Dans cette étape, nous utiliserons Amazon RDS pour créer une instance de base de données MySQL avec une classe d’instance DB db.t2.micro, 20 Go de stockage et automatisé Sauvegardes activées avec une période de conservation d’un jour.

- Ouvre la console de gestion AWS, sélectionnez Base de données à gauche et choisissez RDS pour ouvrir la console Amazon RDS.
  ![image](https://hackmd.io/_uploads/rkkpjNEF-l.png)
- Région sélectionnée
  Dans le coin supérieur droit de la console RDS Amazon, sélectionnez le Région dans laquelle vous souhaitez créer la base de données Exemple.
  ![image](https://hackmd.io/_uploads/HkHe2NVYWe.png)
- Créer une base de données
  Dans la section Créer une base de données, choisissez Créer une base de données.
  ![image](https://hackmd.io/_uploads/Bycf3EVK-e.png)
- Moteur de base de données Select
  Vous avez désormais des options pour sélectionner votre moteur. Pour ce tutoriel, choisissez l’icône MySQL, laissez la valeur par défaut de l’édition et du moteur et sélectionner le modèle de niveau gratuit. Déploiement multi-AZ : Vous devrez payer pour Multi-AZ déploiement. L’utilisation d’un déploiement Multi-AZ provisionne et maintient automatiquement un réplique synchrone en veille dans une zone de disponibilité différente.
  ![image](https://hackmd.io/_uploads/Byx_nEEFZl.png)
- Configurer l’instance de la base de données
  Vous allez maintenant configurer votre instance de base de données. La liste ci-dessous montre le Exemples de réglages que vous pouvez utiliser pour ce tutoriel :

#### Paramètres :

1. Identifiant d’instance de la base de données : Tapez un nom unique pour l’instance de la base de données qui soit unique à votre compte dans la région que vous avez sélectionnée. Pour ce tutoriel, nous Je vais l’appeler **`rds-mysql-10minTutorial`**.
2. Nom d’utilisateur maître : Type a Nom d’utilisateur que vous utiliserez pour vous connecter à votre instance de base de données. Nous j’utiliserai `masterUsername` dans cet exemple.
3. Mot de passe maître : Tapez un mot de passe contenant de 8 à 41 caractères ASCII imprimables (à l’exception de /,", et @) pour votre mot de passe utilisateur principal.
4. Confirmer mot de passe : Retaper Votre mot de passe

![image](https://hackmd.io/_uploads/r1gMTV4Kbe.png)

- Configuration supplémentaire des instances de base de données

#### Spécifications de l’instance :

1. Classe d’instance de base de données : Select db.t2.micro — 1vCPU, 1 Gio BÉLIM. Cela équivaut à 1 Go de mémoire et 1 vCPU. À voir une liste des classes d’instance prises en charge, voir Amazon RDS Pricing.
2. Type de stockage : Sélectionnez l’usage général (SSD). Pour pour plus d’informations sur le stockage, voir Stockage pour Amazon RDS.
3. Stockage alloué : Sélectionner par défaut, 20 Go pour allouer 20 Go de stockage à votre Base de données. Vous pouvez monter à l’échelle jusqu’à un maximum de 64 To avec Amazon RDS pour MySQL.
4. Activez l’auto-scaling du stockage : Si votre charge de travail est cyclique ou imprévisible, vous le ferez Permettre l’autoscaling du stockage pour permettre à Amazon RDS de Augmentez automatiquement votre stockage quand c’est nécessaire. Cette option Cela ne s’applique pas à ce tutoriel.
5. Déploiement multi-AZ : Vous devrez payer pour Déploiement multi-AZ. L’utilisation d’un déploiement Multi-AZ provisionne automatiquement et maintenir une réplique de veille synchrone dans une zone de disponibilité différente. Pour en savoir plus Informations, voir Déploiement à haute disponibilité.
   ![image](https://hackmd.io/_uploads/ByJtpVEFZl.png)

- Configurer la connectivité
  Vous êtes maintenant dans la section Connectivité où vous pouvez fournir les informations dont Amazon RDS a besoin Lance ton instance MySQL. La liste suivante montre les réglages pour notre exemple d’instance de base de données.

#### Connectivité

1. Ressource de calcul : Choisir Ne pas se connecter à un calcul EC2 ressource. Vous pouvez configurer manuellement une connexion à un Ressource de calcul plus tard.
2. Nuage privé virtuel (VPC) : Sélectionner par défaut VPC. Pour plus d’informations sur le VPC, voir Amazon RDS et Amazon Virtual Private Cloud (VPC).

#### Connectivité supplémentaire Configurations

1. Groupe de sous-réseaux : Choisissez le groupe de sous-réseaux par défaut. Pour pour plus d’informations sur les groupes de sous-réseaux, voir Travailler avec la base de données Groupes de sous-réseaux.
2. Accessibilité publique : Choisissez Oui. Cela allouera une adresse IP pour votre instance de base de données donc que vous pouvez vous connecter directement à la base de données depuis votre propre appareil.
3. Groupes de sécurité VPC : Sélectionner Créer une nouvelle sécurité VPC groupe. Cela créera un groupe de sécurité qui permettre la connexion depuis l’adresse IP de l’appareil que vous Sont actuellement en train d’utiliser la base de données créée.
4. Zone de disponibilité : Choisir Sans préférence. Voir Régions et les zones de disponibilité pour plus de détails.
5. Proxy RDS : En utilisant Amazon RDS Proxy, vous pouvez permettre à vos applications de se regrouper et de partager des connexions à la base de données pour améliorer leur capacité à évoluer. Partez le proxy RDS non contrôlé.
6. Transfert : Quitter le mode par défaut valeur de 3306.
   ![image](https://hackmd.io/_uploads/SJvGA44YZx.png)

- Choisir l’option d’authentification
  Amazon RDS prend en compte plusieurs moyens d’authentifier les utilisateurs de bases de données. Choisissez l’authentification par mot de passe parmi la liste des options.

![image](https://hackmd.io/_uploads/rJL8CEEFWl.png)

- Vérification de la surveillance
  Laisser Enable amélioré surveillance sans surveillance pour rester dans le Niveau Libre. Activer une surveillance améliorée vous donnera des indicateurs en temps réel pour le système d’exploitation (OS) sur lequel votre instance de base de données fonctionne.
  ![image](https://hackmd.io/_uploads/By-6AVVKWe.png)
- Définir des options de configuration supplémentaires
  Dans l’Addition Section Configurations :

#### Options de base de données

1. Nom de la base de données : Entrez un Nom de la base de données qui compte de 1 à 64 caractères alphanumériques. Si vous ne donnez pas de nom, Amazon RDS ne le fera pas automatiquement Créez une base de données sur l’instance que vous créez.
2. Groupe de paramètres de la base de données : Leave la valeur par défaut. Pour plus d’informations, voir Travailler avec la base de données Groupes de paramètres.
3. Groupe d’options : Quitter le valeur par défaut. Amazon RDS utilise des groupes d’options pour activer et Configurez des fonctionnalités supplémentaires. Pour plus d’informations, voir Travailler avec des groupes d’options.
4. Chiffrement : Cette option n’est pas disponible dans le palier gratuit. Pour plus d’informations,
5. Remplaçant
   - Période de rétention de secours : Vous pouvez choisir le nombre de jours pour conserver la sauvegarde Prends. Pour ce tutoriel, définissez cette valeur à 1 Jour.
   - Fenêtre de sauvegarde : Utilisez le par défaut of Non Préférence.
6. Mise à niveau de la version automatique mineure : Sélectionner Activer la version mineure automatique Mise à jour pour recevoir des mises à jour automatiques lorsqu’elles devenir disponible.
7. Fenêtre de maintenance : Sélectionner Pas de préférence.
8. Protection contre la suppression : Désactiver Activer la protection contre la suppression pour Ce tutoriel. Lorsque cette option est activée, vous êtes empêché de l’utiliser supprimer accidentellement la base de données
   ![image](https://hackmd.io/_uploads/HJ8RkH4FZe.png)

### Étape 2 : Télécharger un client SQL

Une fois la création de l’instance de base de données terminée et le statut Modifications à la disponibilité, vous pouvez se connecter à une base de données sur l’instance de la base de données en utilisant n’importe quel SQL standard client. À cette étape, nous allons télécharger MySQL Workbench, qui est un un client SQL populaire.

### Étape 3 : Connectez-vous à la base de données MySQL

Dans cette étape, nous allons nous connecter à la base de données que vous avez créée en utilisant Établi MySQL.

- Lancer MySQL Workbench
  Lance l’application MySQL Workbench et va dans la base de données > Connecter à Base de données (Ctrl+U) depuis la barre de menu.
  ![image](https://hackmd.io/_uploads/BJ55gSVKWx.png)

#### Spécifier les options de connexion

1. Une boîte de dialogue apparaît. Voici les éléments suivants :
2. Nom d’hôte : Vous pouvez trouver votre nom d’hôte sur la console Amazon RDS comme montré sur la capture d’écran.
3. Port : La valeur par défaut Ça devrait être 3306.
4. Nom d’utilisateur : Tapez le nom d’utilisateur que vous avez créé pour la base de données Amazon RDS. Dans ce tutoriel, c’est 'masterUsername'.
5. Mot de passe : Choisir Stocker dans le coffre-fort (ou Stocker dans le porte-clés sur MacOS) et saisissez le mot de passe que vous avez utilisé lors de la création de la base de données RDS Amazon.
   ![image](https://hackmd.io/_uploads/ryOkbBNtZe.png)

- Vérification de la connexion à la base de données
  Vous êtes maintenant connecté à la base de données ! Sur l’établi MySQL, vous On verra divers objets schéma disponibles dans la base de données. Maintenant toi peut créer des tables, insérer des données et lancer des requêtes.

![image](https://hackmd.io/_uploads/rJFGWBEKbe.png)

### Étape 4 : Supprimer l’instance de la base de données

Vous pouvez facilement supprimer l’instance MySQL de la base de données depuis le RDS Amazon console. Il est recommandé de supprimer les instances où vous n’êtes pas Plus longtemps pour ne pas être facturé sans cesse.

- Choisissez l’instance de la base de données
  Retourne à la console RDS d’Amazon. Sélectionnez Bases de données, sélectionnez l’instance que vous souhaitez supprimer, puis sélectionnez Supprimer dans le menu déroulant Actions.

![image](https://hackmd.io/_uploads/rko8bH4F-x.png)

- Confirmer la suppression de l’instance
  On vous demande de créer un instantané final et de confirmer le Suppression. Pour notre exemple, ne créez pas un instantané final, reconnaissez que vous souhaitez supprimer l’instance, puis choisissez Supprimer.

![image](https://hackmd.io/_uploads/HJ-KZSEtbl.png)

## Automatisation : AWS CLI

## Remarque

Cette automatisation doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

```PowerShell
<#
.SYNOPSIS
    Script de deploiement RDS MySQL Professionnel - Configuration Cout Zero.
    Desactive : Backups, Snapshots, Monitoring, Performance Insights, Multi-AZ.
.NOTES
    Version : 1.0
#>

# --- 1. CONFIGURATION ---
$Config = @{
    DBInstanceId  = "db-lab-free-pro"
    DBName        = "inventory_db"
    MasterUser    = "admin_root"
    MasterPass    = "WPAdminSecure2026"  # Pas de caracteres speciaux types @ ou !
    Region        = "eu-west-3"
    InstanceClass = "db.t3.micro"
    Storage       = 20
    SGName        = "rds-strict-access-sg"
}

try {
    Write-Host "`n[START] Debut du deploiement RDS (Mode Strict Free Tier)" -ForegroundColor Cyan
    Write-Host "--------------------------------------------------------"

    # --- 2. GESTION DU RESEAU ET SECURITE ---
    $MyIp = (Invoke-RestMethod http://checkip.amazonaws.com).Trim() + "/32"
    Write-Host "[-] Filtrage reseau : Autorisation exclusive pour l'IP $MyIp" -ForegroundColor Yellow

    # Recuperation du VPC par defaut
    $VpcId = (aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text --region $Config.Region)

    # Creation du Security Group
    $SGId = (aws ec2 describe-security-groups --filters "Name=group-name,Values=$($Config.SGName)" --query "SecurityGroups[0].GroupId" --output text --region $Config.Region)

    if ($SGId -eq "None" -or !$SGId) {
        $SGId = (aws ec2 create-security-group --group-name $Config.SGName --description "Accès MySQL restreint" --vpc-id $VpcId --query "GroupId" --output text --region $Config.Region)
        aws ec2 authorize-security-group-ingress --group-id $SGId --protocol tcp --port 3306 --cidr $MyIp --region $Config.Region
        Write-Host "[OK] Security Group cree : $SGId" -ForegroundColor Green
    } else {
        Write-Host "[!] Security Group existant utilise : $SGId" -ForegroundColor Magenta
    }

    # --- 3. CREATION DE L'INSTANCE RDS ---
    Write-Host "[-] Lancement de l'instance RDS MySQL..." -ForegroundColor Yellow

    $CheckRDS = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --region $Config.Region 2>$null)

    if (!$CheckRDS) {
        aws rds create-db-instance `
            --db-instance-identifier $Config.DBInstanceId `
            --db-name $Config.DBName `
            --engine mysql `
            --db-instance-class $Config.InstanceClass `
            --allocated-storage $Config.Storage `
            --storage-type gp2 `
            --master-username $Config.MasterUser `
            --master-user-password $Config.MasterPass `
            --vpc-security-group-ids $SGId `
            --publicly-accessible `
            --no-multi-az `
            --backup-retention-period 0 `
            --monitoring-interval 0 `
            --no-enable-performance-insights `
            --auto-minor-version-upgrade `
            --region $Config.Region | Out-Null
    }

    # --- 4. ATTENTE ACTIVE ---
    Write-Host "[-] Attente de la mise en ligne (Status: Available)..." -NoNewline
    $GlobalTimer = [System.Diagnostics.Stopwatch]::StartNew()

    do {
        Write-Host "." -NoNewline
        Start-Sleep -Seconds 30
        $Status = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --query "DBInstances[0].DBInstanceStatus" --output text --region $Config.Region 2>$null)
        if (!$Status) { $Status = "creating" }
    } while ($Status -ne "available")

    $RDS_Host = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --query "DBInstances[0].Endpoint.Address" --output text --region $Config.Region)

    # --- 5. RESUME FINAL ---
    Write-Host "`n`n[SUCCESS] Instance operationnelle." -ForegroundColor Green
    Write-Host "========================================================"
    Write-Host " ENDPOINT     : $RDS_Host" -ForegroundColor Cyan
    Write-Host " DATABASE     : $($Config.DBName)"
    Write-Host " UTILISATEUR  : $($Config.MasterUser)"
    Write-Host " MOT DE PASSE : $($Config.MasterPass)"
    Write-Host " TEMPS TOTAL  : $([math]::Round($GlobalTimer.Elapsed.TotalMinutes, 2)) min"
    Write-Host "========================================================"

} catch {
    Write-Host "`n[ERREUR] Une erreur est survenue : $($_.Exception.Message)" -ForegroundColor Red
}
```

- Copie le code dans un fichier nommé `DeployRDS.ps1`.
- Ouvre PowerShell en mode Administrateur.
- Exécute : `.\DeployRDS.ps1`.
- Vérification de l'état
  Une fois lancé, tu peux vérifier si l'instance est prête et récupérer l'URL de connexion (Endpoint)

```PowerShell
Get-RDSDBInstance -DBInstanceIdentifier $DBInstanceId | Select-Object DBInstanceStatus, Endpoint
```

### Suppression

```PowerShell
<#
.SYNOPSIS
    Script de suppression complete pour le Lab RDS.
    Supprime l'instance RDS (sans snapshot) et le Security Group associe.
.NOTES
    Auteur: DSPI-TECH
    Version: 1.0
#>

# --- 1. CONFIGURATION ---
$Config = @{
    DBInstanceId = "db-lab-free-pro"
    SGName       = "rds-strict-access-sg"
    Region       = "eu-west-3"
}

try {
    Write-Host "`n[START] Debut du nettoyage des ressources..." -ForegroundColor Cyan
    Write-Host "--------------------------------------------------------"

    # --- 2. SUPPRESSION DE L'INSTANCE RDS ---
    # Verification de l'existence avant suppression
    $RDSStatus = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --query "DBInstances[0].DBInstanceStatus" --output text --region $Config.Region 2>$null)

    if ($RDSStatus -and $RDSStatus -ne "None") {
        Write-Host "[-] Suppression de l'instance RDS : $($Config.DBInstanceId)..." -ForegroundColor Yellow

        # On force la suppression sans Snapshot final et on supprime les backups automatises
        aws rds delete-db-instance `
            --db-instance-identifier $Config.DBInstanceId `
            --skip-final-snapshot `
            --delete-automated-backups `
            --region $Config.Region | Out-Null

        Write-Host "[OK] Commande de suppression envoyee (Statut actuel: $RDSStatus)." -ForegroundColor Green
    } else {
        Write-Host "[i] L'instance RDS n'existe pas ou est deja supprimee." -ForegroundColor Gray
    }

    # --- 3. ATTENTE DE SUPPRESSION TOTALE ---
    # Un Security Group ne peut pas etre supprime tant que la DB utilise encore son interface reseau
    Write-Host "[-] Attente de la liberation des ressources (5-8 min)..." -NoNewline
    do {
        Write-Host "." -NoNewline
        Start-Sleep -Seconds 30
        $CheckRDS = (aws rds describe-db-instances --db-instance-identifier $Config.DBInstanceId --region $Config.Region 2>$null)
    } while ($CheckRDS)

    Write-Host "`n[OK] Instance RDS supprimee avec succes." -ForegroundColor Green

    # --- 4. SUPPRESSION DU SECURITY GROUP ---
    Write-Host "[-] Recherche du Security Group : $($Config.SGName)..." -ForegroundColor Yellow

    $SGId = (aws ec2 describe-security-groups --filters "Name=group-name,Values=$($Config.SGName)" --query "SecurityGroups[0].GroupId" --output text --region $Config.Region)

    if ($SGId -and $SGId -ne "None") {
        # On tente la suppression
        aws ec2 delete-security-group --group-id $SGId --region $Config.Region
        Write-Host "[OK] Security Group $SGId supprime." -ForegroundColor Green
    } else {
        Write-Host "[i] Security Group introuvable ou deja supprime." -ForegroundColor Gray
    }

    Write-Host "`n[SUCCESS] Environnement totalement nettoye. Zero cout genere." -ForegroundColor Cyan
    Write-Host "========================================================"

} catch {
    Write-Host "`n[ERREUR] Un probleme est survenu lors du nettoyage : $($_.Exception.Message)" -ForegroundColor Red
}
```

### Vérification

```PowerShell
<#
.SYNOPSIS
    Verifie les ressources RDS et EC2 actives pour eviter la facturation.
.NOTES
    Version : 1.0
#>

$Region = "eu-west-3"

Write-Host "`n--- VERIFICATION DES RESSOURCES ACTIVES ($Region) ---" -ForegroundColor Cyan

# 1. Verifier les instances RDS
Write-Host "[-] Verification des instances RDS..." -ForegroundColor Gray
$Dbs = (aws rds describe-db-instances --query "DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Engine]" --output table --region $Region 2>$null)

if ($Dbs) {
    Write-Host "[!] ALERTE : Il reste des ressources RDS actives !" -ForegroundColor Red
    $Dbs
} else {
    Write-Host "[OK] Aucune instance RDS trouvee." -ForegroundColor Green
}

# 2. Verifier les Snapshots RDS (Souvent la cause de frais caches)
Write-Host "[-] Verification des Snapshots RDS..." -ForegroundColor Gray
$Snapshots = (aws rds describe-db-snapshots --query "DBSnapshots[*].[DBSnapshotIdentifier,Status]" --output table --region $Region 2>$null)

if ($Snapshots) {
    Write-Host "[!] ATTENTION : Il reste des Snapshots RDS (Facturation possible)." -ForegroundColor Yellow
    $Snapshots
} else {
    Write-Host "[OK] Aucun Snapshot RDS trouve." -ForegroundColor Green
}

# 3. Verifier les Instances EC2 (en cours ou stoppees)
Write-Host "[-] Verification des instances EC2..." -ForegroundColor Gray
$Ec2 = (aws ec2 describe-instances --filters "Name=instance-state-name,Values=running,stopped,pending" --query "Reservations[*].Instances[*].[InstanceId,State.Name]" --output table --region $Region 2>$null)

if ($Ec2) {
    Write-Host "[!] ALERTE : Il reste des instances EC2 (Facturation IP possible) !" -ForegroundColor Red
    $Ec2
} else {
    Write-Host "[OK] Aucune instance EC2 active ou stoppee trouvee." -ForegroundColor Green
}

# 4. Verifier les Groupes de Securite (Hors defaut)
Write-Host "[-] Verification des Security Groups..." -ForegroundColor Gray
$Sgs = (aws ec2 describe-security-groups --query "SecurityGroups[?GroupName!='default'].[GroupName,GroupId]" --output table --region $Region 2>$null)

if ($Sgs) {
    Write-Host "[i] Information : Groupes de securite personnalises presents (Gratuit)." -ForegroundColor Gray
    $Sgs
} else {
    Write-Host "[OK] Aucun Security Group personnalise trouve." -ForegroundColor Green
}

Write-Host "`n--- FIN DE L'AUDIT ---" -ForegroundColor Cyan
```
