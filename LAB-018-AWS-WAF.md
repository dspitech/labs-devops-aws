# LAB : Commencer à AWS WAF utiliser la nouvelle expérience de console

## Objectif du lab

Ce lab permet de **protéger une API REST avec AWS WAF** en utilisant PowerShell et AWS CLI.  
À la fin, vous serez capable de :

- Créer une **API Gateway de test**.
- Déployer un **Web ACL** pour la sécurisation.
- Ajouter des **règles préconfigurées**.
- Lier le Web ACL à l’API.
- Ajouter des protections avancées (IP Sets & Rate Limiting).
- Nettoyer toutes les ressources pour éviter les coûts.

---

## Description

AWS WAF (Web Application Firewall) permet de **protéger vos applications web contre les attaques courantes** : injection SQL, scripts intersites, bots, etc.  
Ce lab illustre la création et l’association d’un Web ACL sur une API Gateway régionale, ainsi que la personnalisation via des listes IP.

## Remarque

Ce lab doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

## Étape 1 : Configuration et Création de la Ressource

```PowerShell
$region = "eu-west-3" # Région Paris

# 1. Créer l'API de test
$api = aws apigateway create-rest-api --name "MonApp-API" --region $region | ConvertFrom-Json
$apiId = $api.id

# 2. Déployer un stage "prod" pour obtenir un ARN valide
$rootId = (aws apigateway get-resources --rest-api-id $apiId --region $region | ConvertFrom-Json).items[0].id
aws apigateway put-method --rest-api-id $apiId --resource-id $rootId --http-method GET --authorization-type "NONE" --region $region
aws apigateway create-deployment --rest-api-id $apiId --stage-name "prod" --region $region

# 3. ARN de la ressource à protéger
$resourceArn = "arn:aws:apigateway:$region::/restapis/$apiId/stages/prod"
```

![image](https://hackmd.io/_uploads/BJdeDJwKWx.png)
![image](https://hackmd.io/_uploads/rkZzwJwYbl.png)

## Étape 2 : Création du Pack de Protection (Web ACL)

```PowerShell
#
aws wafv2 create-web-acl --% --name "ESTIAMParis" --scope REGIONAL --default-action Allow={} --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName="ESTIAMParis" --region eu-west-3
```

![image](https://hackmd.io/_uploads/S1L9wJPtZl.png)
![image](https://hackmd.io/_uploads/BJvNuJvKZx.png)

## Étape 3 : Ajouter les "Protections Initiales" (Règles Recommandées)

Dans ta console, clique sur ESTIAMParis.

- Clique sur l'onglet Règles (Rules), puis Ajouter des règles (Add rules).
- Sélectionne Groupes de règles gérés par AWS (Add managed rule groups).
- Cherche Free managed rule groups et active Core rule set.
- Clique sur Sauvegarder.
- ![image](https://hackmd.io/_uploads/rJcucyDFbg.png)
  ![image](https://hackmd.io/_uploads/HJDYqJvtWx.png)
  ![image](https://hackmd.io/_uploads/Hk6ccJwt-l.png)
  ![image](https://hackmd.io/_uploads/BJYLikDFZl.png)
  ![image](https://hackmd.io/_uploads/BkKTjkDFWl.png)
  ![image](https://hackmd.io/_uploads/r19AsywKZe.png)
  ![image](https://hackmd.io/_uploads/ryDlhkvtZl.png)

## Étape 4 : Associer la Ressource (Lier le Pack à l'API)

```PowerShell
aws --% wafv2 associate-web-acl --web-acl-arn "arn:aws:wafv2:eu-west-3:157125373775:regional/webacl/ESTIAMParis/2f29c5d7-ae42-4d34-b545-8e31c1b471e5" --resource-arn "arn:aws:apigateway:eu-west-3::/restapis/gsutvxyj7c/stages/prod" --region eu-west-3
```

![image](https://hackmd.io/_uploads/rySe6kwFWl.png)
![image](https://hackmd.io/_uploads/ryl8ayPY-x.png)

## Étape 5 : Personnalisation (IP Sets & Rate Limiting)

```PowerShell
# Création de la liste IP (IP Set)
$ipSet = aws --% wafv2 create-ip-set --name "MaListeAutorisee" --scope REGIONAL --ip-address-version IPV4 --addresses "1.2.3.4/32" --region eu-west-3 | ConvertFrom-Json

# récupérer l'ARN :
$ipSet.Summary.ARN
```

![image](https://hackmd.io/_uploads/H1StpJwFWx.png)
![image](https://hackmd.io/_uploads/Ske101vYZl.png)
![image](https://hackmd.io/_uploads/rJfe01vtbe.png)

## Étape 6 : Nettoyage

```PowerShell
Write-Host "--- NETTOYAGE AUTOMATIQUE EN COURS ---" -ForegroundColor Cyan

# 1. SUPPRIMER L'API GATEWAY (gsutvxyj7c)
$api = aws apigateway get-rest-apis --region eu-west-3 --query "items[?name=='MonApp-API'].id" --output text
if ($api) {
    aws apigateway delete-rest-api --rest-api-id $api --region eu-west-3
    Write-Host " API Gateway supprimée ($api)" -ForegroundColor Yellow
}

# 2. DISSOCIER ET SUPPRIMER LE WEB ACL (ESTIAMParis)
$webAcl = aws wafv2 list-web-acls --scope REGIONAL --region eu-west-3 --query "WebACLs[?Name=='ESTIAMParis'] | [0]" | ConvertFrom-Json
if ($webAcl) {
    # Dissociation forcée (au cas où)
    $resourceArn = "arn:aws:apigateway:eu-west-3::/restapis/$api/stages/prod"
    aws wafv2 disassociate-web-acl --resource-arn $resourceArn --region eu-west-3 2>$null

    # Suppression
    $token = (aws wafv2 get-web-acl --name "ESTIAMParis" --scope REGIONAL --id $webAcl.Id --region eu-west-3 --query LockToken --output text)
    aws wafv2 delete-web-acl --name "ESTIAMParis" --scope REGIONAL --id $webAcl.Id --lock-token $token --region eu-west-3
    Write-Host " Web ACL 'ESTIAMParis' supprimé" -ForegroundColor Yellow
}

# 3. SUPPRIMER L'IP SET (MaListeAutorisee)
$ipSet = aws wafv2 list-ip-sets --scope REGIONAL --region eu-west-3 --query "IPSets[?Name=='MaListeAutorisee'] | [0]" | ConvertFrom-Json
if ($ipSet) {
    $token = (aws wafv2 get-ip-set --name "MaListeAutorisee" --scope REGIONAL --id $ipSet.Id --region eu-west-3 --query LockToken --output text)
    aws wafv2 delete-ip-set --name "MaListeAutorisee" --scope REGIONAL --id $ipSet.Id --lock-token $token --region eu-west-3
    Write-Host " IP Set 'MaListeAutorisee' supprimé" -ForegroundColor Yellow
}

Write-Host "--- TOUT EST PROPRE ! ---" -ForegroundColor Green
```

![image](https://hackmd.io/_uploads/S1k6blvtZx.png)
![image](https://hackmd.io/_uploads/ByV7GxDt-l.png)
