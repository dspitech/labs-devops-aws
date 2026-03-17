# LAB : Déployer un site web statique sur S3

## Description

Ce lab montre comment :

- créer un **bucket S3**
- activer le **hosting web statique**
- uploader les fichiers **HTML/CSS/JS**
- rendre le site **accessible publiquement**

**S3** permet de déployer un site web **sans serveur**, à faible coût et très rapidement.

---

## Objectifs

- Créer un **bucket S3** adapté au **hosting web**
- Configurer le **site statique**
- Déployer les **fichiers du site**
- Accéder au site via l’**URL publique**

## Remarque

Ce lab doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

## Création du Bucket (Région Paris)

```PowerShell
aws s3api create-bucket `
  --bucket estiam.com `
  --region eu-west-3 `
  --create-bucket-configuration LocationConstraint=eu-west-3
```

![image](https://hackmd.io/_uploads/SkqJAsHKZg.png)
![image](https://hackmd.io/_uploads/H1FZ0iHFWl.png)

## Configuration de l'accès public

```PowerShell
aws s3api put-public-access-block `
  --bucket estiam.com `
  --public-access-block-configuration BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

![image](https://hackmd.io/_uploads/BynE0sSFWx.png)
![image](https://hackmd.io/_uploads/rkBD0sBYWl.png)

## Activation du mode "Site Web"

```PowerShell
aws s3 website s3://estiam.com/ `
  --index-document index.html `
  --error-document error.html
```

![image](https://hackmd.io/_uploads/HyoF0oHtWx.png)
![image](https://hackmd.io/_uploads/Hykb12SF-g.png)

## Application de la Politique de Lecture (Bucket Policy)

```PowerShell
$policy = @"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::estiam.com/*"
    }
  ]
}
"@
$policy | Out-File -FilePath policy.json -Encoding ascii

aws s3api put-bucket-policy --bucket estiam.com --policy file://policy.json
```

![image](https://hackmd.io/_uploads/BJKzyhBtWe.png)
![image](https://hackmd.io/_uploads/H1ALy2SYZe.png)

## Envoi des fichiers (Upload)

```Bash
# Cloner le projet depuis Github
git clone https://github.com/dspitech/Configuring-a-Static-Website-on-Amazon-S3.git

# Se positionner dans le dossier du projet
aws s3 sync "D:\AWS\Labs\Bucket-S3\Site-Web-AWS" s3://estiam.com --exclude "policy.json"
```

> Changer le chemin : `D:\AWS\Labs\Bucket-S3\Site-Web-AWS` et metter le chemin de vos fichiers du site
>
> ![image](https://hackmd.io/_uploads/ByRPl3BtWl.png)
> ![image](https://hackmd.io/_uploads/S1Z5x3rFZl.png)

## Obtenir l'URL

```PowerShell
$BucketName = "estiam.com"
$Region = "eu-west-3"
Write-Host "--- TON SITE EST EN LIGNE ---" -ForegroundColor Green
Write-Host "URL : http://$BucketName.s3-website-$Region.amazonaws.com" -ForegroundColor Cyan
```

![image](https://hackmd.io/_uploads/Hyone3rtbe.png)
![image](https://hackmd.io/_uploads/Bymb-3SFbl.png)

## Nettoyage complet

```PowerShell
aws s3 rb s3://estiam.com --force
```

![image](https://hackmd.io/_uploads/BJqVb2BFbg.png)

![image](https://hackmd.io/_uploads/rkbUWnrt-x.png)
