# LAB : AWS Calculator

## Objectif du lab

Ce lab montre comment **simuler et calculer les coûts AWS** pour différentes architectures via PowerShell :

- Architecture **monolithique**
- Architecture **microservices**
- Architecture **événementielle / serverless**

Vous pourrez ainsi comparer **USD vs EUR**, et estimer le coût mensuel et annuel avec TVA.

---

## Description

AWS Pricing Calculator fournit des estimations basées sur :

- Type de service (Compute, Storage, Database, Networking…)
- Quantité et usage (heures, Go, requêtes)
- Région de déploiement
- Conversion monétaire et taxes locales

Ce lab crée des objets PowerShell pour chaque service et effectue :

1. **Somme mensuelle et annuelle en USD**
2. **Conversion en EUR avec TVA**
3. **Affichage formaté pour comparaison**

## Remarque

Ce lab doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

## Architetcure monolithique

![image](https://hackmd.io/_uploads/BJrkd285Wl.png)

```PowerShell
# --- TARIFS Architecture monolithique ---
$ArchitectureServices = @(
    [PSCustomObject]@{ Service = "VPC & Subnets";        Type = "Reseau";    Note = "Inclus dans AWS";      Mois = 0.00;  Annee = 0.00 }
    [PSCustomObject]@{ Service = "IAM & Security Groups"; Type = "Securite";  Note = "Service Gratuit";      Mois = 0.00;  Annee = 0.00 }
    [PSCustomObject]@{ Service = "Elastic IP (EIP)";     Type = "Reseau";    Note = "Gratuit si attachee";  Mois = 0.00;  Annee = 0.00 }
    [PSCustomObject]@{ Service = "EC2 (t3.medium)";      Type = "Compute";   Note = "0.0472 $/heure";       Mois = 34.46; Annee = 413.52 }
    [PSCustomObject]@{ Service = "RDS MySQL (db.t3.micro)";Type = "Database";  Note = "0.038 $/heure";        Mois = 27.74; Annee = 332.88 }
    [PSCustomObject]@{ Service = "EBS Storage (30GB gp3)";Type = "Stockage";  Note = "0.093 $/Go";           Mois = 2.79;  Annee = 33.48 }
    [PSCustomObject]@{ Service = "S3 (50GB Standard)";   Type = "Stockage";  Note = "0.023 $/Go";           Mois = 1.15;  Annee = 13.80 }
    [PSCustomObject]@{ Service = "AWS Secrets Manager";  Type = "Securite";  Note = "0.40 $ / secret";      Mois = 0.40;  Annee = 4.80 }
    [PSCustomObject]@{ Service = "Route 53 (1 Zone)";    Type = "DNS";       Note = "0.50 $ / zone";        Mois = 0.50;  Annee = 6.00 }
    [PSCustomObject]@{ Service = "CloudFront (Estim.)";  Type = "CDN";       Note = "Usage (10GB/mo)";      Mois = 0.85;  Annee = 10.20 }
)

# --- CALCULS ---
$TotalMoisUSD = ($ArchitectureServices | Measure-Object -Property Mois -Sum).Sum
$TotalAnUSD   = ($ArchitectureServices | Measure-Object -Property Annee -Sum).Sum
$TauxChange   = 0.92
$TVA          = 1.20
$TotalMoisEUR_TTC = ($TotalMoisUSD * $TauxChange) * $TVA

# --- AFFICHAGE ---
Clear-Host
$line = "=" * 78
$thin = "-" * 78

Write-Host "`n$line" -ForegroundColor Cyan
Write-Host "   RAPPORT DE COUTS AWS - ARCHITECTURE WORDPRESS (PARIS)" -ForegroundColor Cyan
Write-Host "   Source: AWS Pricing Calculator | Devise: USD ($)" -ForegroundColor Cyan
Write-Host $line -ForegroundColor Cyan

# Affichage du tableau formaté
$ArchitectureServices | Format-Table @{Expression="Service";Width=25}, @{Expression="Type";Width=12}, @{Expression="Note";Width=20}, @{Expression="Mois";Width=8;FormatString="{0:N2} $"}, @{Expression="Annee";Width=10;FormatString="{0:N2} $"}

Write-Host $thin
Write-Host " TOTAL MENSUEL HT (USD) : $([Math]::Round($TotalMoisUSD, 2)) $" -ForegroundColor White
Write-Host " TOTAL ANNUEL  HT (USD) : $([Math]::Round($TotalAnUSD, 2)) $" -ForegroundColor White
Write-Host $thin
Write-Host " TOTAL ESTIME TTC (EUR) : $([Math]::Round($TotalMoisEUR_TTC, 2)) EUR / mois" -ForegroundColor Yellow
Write-Host " (Conversion: 1 USD = 0.92 EUR | TVA: 20%)" -ForegroundColor Gray
Write-Host "$line`n" -ForegroundColor Cyan
```

![image](https://hackmd.io/_uploads/Hk74waAFbl.png)

## Architecture Microservice

![image](https://hackmd.io/_uploads/H1G-u3LqWg.png)

```PowerShell
# ---  TARIFS Architecture Microservice ---
$ArchitectureServices = @(
    [PSCustomObject]@{ Service = "VPC, Subnets & IGW";    Type = "Reseau";    Note = "Inclus dans AWS";      Mois = 0.00;   Annee = 0.00 }
    [PSCustomObject]@{ Service = "IAM & Security Groups"; Type = "Securite";  Note = "Service Gratuit";      Mois = 0.00;   Annee = 0.00 }
    [PSCustomObject]@{ Service = "Route 53 (1 Zone)";    Type = "DNS";       Note = "0.50 $ / zone";        Mois = 0.50;   Annee = 6.00 }
    [PSCustomObject]@{ Service = "Application Load Bal.";Type = "Reseau";    Note = "ALB + LCU usage";      Mois = 22.26;  Annee = 267.12 }
    [PSCustomObject]@{ Service = "ECS Fargate (2 vCPU)"; Type = "Compute";   Note = "Fargate vCPU/RAM";     Mois = 48.18;  Annee = 578.16 }
    [PSCustomObject]@{ Service = "NAT Gateway (1 unit)"; Type = "Reseau";    Note = "0.045 $/heure";        Mois = 32.85;  Annee = 394.20 }
    [PSCustomObject]@{ Service = "RDS MySQL (db.t3.micro)";Type = "Database";  Note = "Multi-AZ";             Mois = 27.74;  Annee = 332.88 }
    [PSCustomObject]@{ Service = "ElastiCache (Redis)";  Type = "Cache";     Note = "Primary + Standby";    Mois = 25.42;  Annee = 305.04 }
    [PSCustomObject]@{ Service = "EFS (Standard 50GB)";  Type = "Stockage";  Note = "0.33 $/Go (Paris)";    Mois = 16.50;  Annee = 198.00 }
    [PSCustomObject]@{ Service = "S3 (50GB Standard)";   Type = "Stockage";  Note = "0.023 $/Go";           Mois = 1.15;   Annee = 13.80 }
    [PSCustomObject]@{ Service = "AWS WAF (Base)";       Type = "Securite";  Note = "Web ACL + Rules";      Mois = 6.00;   Annee = 72.00 }
    [PSCustomObject]@{ Service = "AWS Secrets Manager";  Type = "Securite";  Note = "0.40 $ / secret";      Mois = 0.40;   Annee = 4.80 }
    [PSCustomObject]@{ Service = "CloudFront (Estim.)";  Type = "CDN";       Note = "Usage (10GB/mo)";      Mois = 0.85;   Annee = 10.20 }
)

# --- CALCULS ---
$TotalMoisUSD = ($ArchitectureServices | Measure-Object -Property Mois -Sum).Sum
$TotalAnUSD   = ($ArchitectureServices | Measure-Object -Property Annee -Sum).Sum
$TauxChange   = 0.92
$TVA          = 1.20
$TotalMoisEUR_TTC = ($TotalMoisUSD * $TauxChange) * $TVA

# --- AFFICHAGE ---
Clear-Host
$line = "=" * 85
$thin = "-" * 85

Write-Host "`n$line" -ForegroundColor Cyan
Write-Host "   RAPPORT DE COUTS AWS - ARCHITECTURE COMPLETE (PARIS)" -ForegroundColor Cyan
Write-Host "   Source: AWS Pricing Calculator | Region: eu-west-3 | Devise: USD ($)" -ForegroundColor Cyan
Write-Host $line -ForegroundColor Cyan

# Affichage du tableau
$ArchitectureServices | Format-Table `
    @{Expression="Service";Label="Service AWS";Width=28}, `
    @{Expression="Type";Label="Categorie";Width=12}, `
    @{Expression="Note";Label="Details Tarif";Width=22}, `
    @{Expression="Mois";Label="Mois ($)";Width=10;FormatString="{0:N2} $"}, `
    @{Expression="Annee";Label="Annee ($)";Width=10;FormatString="{0:N2} $"}

Write-Host $thin
Write-Host " TOTAL MENSUEL HT (USD) : $([Math]::Round($TotalMoisUSD, 2)) $" -ForegroundColor White
Write-Host " TOTAL ANNUEL  HT (USD) : $([Math]::Round($TotalAnUSD, 2)) $" -ForegroundColor White
Write-Host $thin
Write-Host " TOTAL ESTIME TTC (EUR) : $([Math]::Round($TotalMoisEUR_TTC, 2)) EUR / mois" -ForegroundColor Yellow
Write-Host " (Conversion: 1 USD = 0.92 EUR | TVA: 20%)" -ForegroundColor Gray
Write-Host "$line`n" -ForegroundColor Cyan
```

![image](https://hackmd.io/_uploads/ByPnwaCFWg.png)

## Architecture Événementielle

![image](https://hackmd.io/_uploads/HkVzO38cbl.png)

```PowerShell
# ---  --- TARIFS Architecture Événementielle  ---
# Les tarifs incluent les services Serverless avec un usage standard de base.

$ArchitectureServices = @(
    [PSCustomObject]@{ Service = "VPC, Subnets & IGW";    Type = "Reseau";    Note = "Inclus dans AWS";      Mois = 0.00;   Annee = 0.00 }
    [PSCustomObject]@{ Service = "IAM & Security Groups"; Type = "Securite";  Note = "Service Gratuit";      Mois = 0.00;   Annee = 0.00 }
    [PSCustomObject]@{ Service = "Route 53 (1 Zone)";    Type = "DNS";       Note = "0.50 $ / zone";        Mois = 0.50;   Annee = 6.00 }
    [PSCustomObject]@{ Service = "Application Load Bal.";Type = "Reseau";    Note = "ALB + LCU usage";      Mois = 22.26;  Annee = 267.12 }
    [PSCustomObject]@{ Service = "ECS Fargate (2 vCPU)"; Type = "Compute";   Note = "Fargate vCPU/RAM";     Mois = 48.18;  Annee = 578.16 }
    [PSCustomObject]@{ Service = "NAT Gateway (1 unit)"; Type = "Reseau";    Note = "0.045 $/heure";        Mois = 32.85;  Annee = 394.20 }
    [PSCustomObject]@{ Service = "RDS MySQL (db.t3.micro)";Type = "Database";  Note = "Multi-AZ";             Mois = 27.74;  Annee = 332.88 }
    [PSCustomObject]@{ Service = "ElastiCache (Redis)";  Type = "Cache";     Note = "Primary + Standby";    Mois = 25.42;  Annee = 305.04 }
    [PSCustomObject]@{ Service = "EFS (Standard 50GB)";  Type = "Stockage";  Note = "0.33 $/Go (Paris)";    Mois = 16.50;  Annee = 198.00 }
    [PSCustomObject]@{ Service = "S3 (50GB Standard)";   Type = "Stockage";  Note = "0.023 $/Go";           Mois = 1.15;   Annee = 13.80 }
    [PSCustomObject]@{ Service = "AWS WAF (Base)";       Type = "Securite";  Note = "Web ACL + Rules";      Mois = 6.00;   Annee = 72.00 }
    [PSCustomObject]@{ Service = "AWS Secrets Manager";  Type = "Securite";  Note = "0.40 $ / secret";      Mois = 0.40;   Annee = 4.80 }
    [PSCustomObject]@{ Service = "CloudFront (Estim.)";  Type = "CDN";       Note = "Usage (10GB/mo)";      Mois = 0.85;   Annee = 10.20 }
    [PSCustomObject]@{ Service = "Lambda (1M req/mo)";   Type = "Serverless";Note = "Free Tier inclus";     Mois = 0.20;   Annee = 2.40 }
    [PSCustomObject]@{ Service = "SQS & SNS";            Type = "Messaging"; Note = "Usage standard";       Mois = 0.10;   Annee = 1.20 }
    [PSCustomObject]@{ Service = "EventBridge";          Type = "Serverless";Note = "Bus par defaut";       Mois = 0.00;   Annee = 0.00 }
)

# --- CALCULS ---
$TotalMoisUSD = ($ArchitectureServices | Measure-Object -Property Mois -Sum).Sum
$TotalAnUSD   = ($ArchitectureServices | Measure-Object -Property Annee -Sum).Sum
$TauxChange   = 0.92
$TVA          = 1.20
$TotalMoisEUR_TTC = ($TotalMoisUSD * $TauxChange) * $TVA

# --- AFFICHAGE ---
Clear-Host
$line = "=" * 85
$thin = "-" * 85

Write-Host "`n$line" -ForegroundColor Cyan
Write-Host "   RAPPORT DE COUTS AWS - ARCHITECTURE MICROSERVICES PRO (PARIS)" -ForegroundColor Cyan
Write-Host "   Source: AWS Pricing Calculator | Region: eu-west-3 | Devise: USD ($)" -ForegroundColor Cyan
Write-Host $line -ForegroundColor Cyan

# Affichage du tableau formate
$ArchitectureServices | Format-Table `
    @{Expression="Service";Label="Service AWS";Width=28}, `
    @{Expression="Type";Label="Categorie";Width=12}, `
    @{Expression="Note";Label="Details Tarif";Width=22}, `
    @{Expression="Mois";Label="Mois ($)";Width=10;FormatString="{0:N2} $"}, `
    @{Expression="Annee";Label="Annee ($)";Width=10;FormatString="{0:N2} $"}

Write-Host $thin
Write-Host " TOTAL MENSUEL HT (USD) : $([Math]::Round($TotalMoisUSD, 2)) $" -ForegroundColor White
Write-Host " TOTAL ANNUEL  HT (USD) : $([Math]::Round($TotalAnUSD, 2)) $" -ForegroundColor White
Write-Host $thin
Write-Host " TOTAL ESTIME TTC (EUR) : $([Math]::Round($TotalMoisEUR_TTC, 2)) EUR / mois" -ForegroundColor Yellow
Write-Host " (Conversion: 1 USD = 0.92 EUR | TVA: 20%)" -ForegroundColor Gray
Write-Host "$line`n" -ForegroundColor Cyan
```

![image](https://hackmd.io/_uploads/B1nHu6CFZg.png)
