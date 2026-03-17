# LAB AWS CloudTrail : Auditing & Compliance

## Objectif

- Créer un **trail CloudTrail** pour enregistrer toutes les actions AWS
- Activer la **journalisation** vers un **bucket S3**
- Auditer les **événements** pour suivre les actions des utilisateurs
- Analyser et visualiser les **logs** pour la conformité

## Remarque

Ce lab doit être réalisé dans **AWS CloudShell**.

## Déploiement

### Créer un bucket S3 pour stocker les logs

```Bash
# Nom du bucket
BUCKET_NAME="devops-cloudtrail-logs-$(date +%s)"

# Création du bucket
aws s3 mb s3://$BUCKET_NAME --region eu-west-3

# Ajouter des tags
aws s3api put-bucket-tagging \
  --bucket $BUCKET_NAME \
  --tagging 'TagSet=[{Key=Environment,Value=Lab},{Key=Project,Value=CloudTrail}]'

# Vérification
aws s3 ls | grep $BUCKET_NAME
```

![image](https://hackmd.io/_uploads/rkMs8o-qbg.png)

![image](https://hackmd.io/_uploads/ryYRLiW5bg.png)

### Crée une policy pour le bucket

```Bash
cat > bucket-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AWSCloudTrailAclCheck20150319",
            "Effect": "Allow",
            "Principal": { "Service": "cloudtrail.amazonaws.com" },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::$BUCKET_NAME"
        },
        {
            "Sid": "AWSCloudTrailWrite20150319",
            "Effect": "Allow",
            "Principal": { "Service": "cloudtrail.amazonaws.com" },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::$BUCKET_NAME/AWSLogs/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        }
    ]
}
EOF
```

![image](https://hackmd.io/_uploads/HktjviZ9-e.png)

### Attache la policy au bucket

```Bash
aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy file://bucket-policy.json
```

![image](https://hackmd.io/_uploads/HyXavoW9bx.png)

### Créer le trail CloudTrail

```Bash
aws cloudtrail create-trail \
    --name DevOpsTrail \
    --s3-bucket-name $BUCKET_NAME \
    --include-global-service-events \
    --is-multi-region-trail \
    --no-enable-log-file-validation
```

![image](https://hackmd.io/_uploads/BJMgusbqWg.png)
![image](https://hackmd.io/_uploads/SJLyYiZcWl.png)

#### Options importantes CloudTrail

- `--include-global-service-events` : capture les **événements de tous les services globaux** (IAM, CloudFront, etc.)
- `--is-multi-region-trail` : capture les **actions dans toutes les régions**

#### Vérifie la création du trail

```Bash
aws cloudtrail describe-trails --trail-name-list DevOpsTrail
```

![image](https://hackmd.io/_uploads/ryGUujZqWg.png)

### Activer la journalisation

```Bash
aws cloudtrail start-logging --name DevOpsTrail
aws s3 ls s3://$BUCKET_NAME --recursive
```

![image](https://hackmd.io/_uploads/BypvuiZ5-g.png)

#### Vérifie que la journalisation est active

```Bash
aws cloudtrail get-trail-status --name DevOpsTrail
```

- `IsLogging: true` → tout est prêt

![image](https://hackmd.io/_uploads/Skd9_sWq-e.png)
![image](https://hackmd.io/_uploads/SkBBFsWqWe.png)

### Faire un test (générer des événements)

Exécute quelques commandes AWS pour créer des événements

```Bash
aws s3 ls
aws iam list-users
```

![image](https://hackmd.io/_uploads/rk-_tj-qZx.png)
![image](https://hackmd.io/_uploads/H1xvFo-9-l.png)

![image](https://hackmd.io/_uploads/S15ycoZcWe.png)

Ces actions seront maintenant capturées par CloudTrail.

### Rechercher des événements pour l’audit

#### Rechercher par utilisateur

```Bash
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=Username,AttributeValue=DspiAdmin \
    --max-results 5
```

Dans AttributeValue=`DspiAdmin` : changer le et mettre votre utisateur.

![image](https://hackmd.io/_uploads/HyM_qoWc-x.png)
![image](https://hackmd.io/_uploads/BJgcqsZ5Zg.png)
![image](https://hackmd.io/_uploads/B1usqsb5-l.png)

#### Rechercher par ressource (exemple : S3)

```Bash
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=ResourceName,AttributeValue=$BUCKET_NAME \
    --max-results 5
```

![image](https://hackmd.io/_uploads/BJ8yjo-c-l.png)
![image](https://hackmd.io/_uploads/HkUZio-5Zl.png)

Tu obtiens les actions qui ont été réalisées, l’heure, l’ARN, et le type d’événement.

### Nettoyage

#### Supprimer le trail CloudTrail

```Bash
# Arrêter la journalisation
aws cloudtrail stop-logging --name DevOpsTrail --region eu-west-3

# Supprimer le trail
aws cloudtrail delete-trail --name DevOpsTrail --region eu-west-3
```

![image](https://hackmd.io/_uploads/rJe0BjoWqbg.png)
![image](https://hackmd.io/_uploads/ByY_oj-9Ze.png)

Vérifie qu’il n’existe plus.

```Bash
aws cloudtrail describe-trails --trail-name-list DevOpsTrail
```

![image](https://hackmd.io/_uploads/SJO9jsW5We.png)

#### Supprimer la policy attachée au bucket et le fichier JSON

```Bash
# Supprimer la policy attachée au bucket
aws s3api delete-bucket-policy --bucket $BUCKET_NAME

# Supprimer le fichier local de policy (optionnel)
rm -f bucket-policy.json
```

![image](https://hackmd.io/_uploads/Hk6bpib5We.png)

#### Supprimer le bucket S3 et son contenu

```Bash
# Supprimer tous les objets du bucket
aws s3 rm s3://$BUCKET_NAME --recursive

# Supprimer le bucket
aws s3 rb s3://$BUCKET_NAME
```

![image](https://hackmd.io/_uploads/BkB2ioW5Zg.png)
![image](https://hackmd.io/_uploads/B1gRss-9-x.png)
![image](https://hackmd.io/_uploads/SJArhoW9Wx.png)

#### Vérification finale

```Bash
# Liste les buckets
aws s3 ls

# Liste les trails
aws cloudtrail describe-trails
```

![image](https://hackmd.io/_uploads/ry_mTi-5-g.png)
