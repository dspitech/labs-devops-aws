# LAB 026 : IAM > OU, Groupes et Utilisateurs

## Objectif

- Créer une organisation AWS structurée par secteur (OU)
- Déployer des groupes IAM et rôles par fonction et secteur
- Créer des utilisateurs types et leur attribuer des permissions via les groupes
- Activer MFA et configurer l’accès CLI
- Appliquer des bonnes pratiques RBAC et gouvernance

## Structure des Directions et Groupes

| Secteur / Direction  | Groupes AWS suggérés | Rôle / Responsabilité                         |
| -------------------- | -------------------- | --------------------------------------------- |
| Direction IT / Cloud | CloudAdmins          | Accès total à l'infrastructure et facturation |
|                      | SecurityAudit        | Lecture seule pour la conformité et les logs  |
| Secteur Industrie    | Indus-Dev            | Gestion ressources Dev (EC2, IoT…)            |
|                      | Indus-Ops            | Maintenance production                        |
| Secteur Finance      | Fin-DataAnalysts     | Accès aux outils Data (S3, Redshift)          |
|                      | Fin-Compliance       | Audit des données financières                 |
| Ressources Humaines  | HR-AppSupport        | Support technique des applications internes   |

## Rôles et Permissions (Modèle RBAC)

| Groupe / Rôle             | Policy AWS              | Description                                             |
| ------------------------- | ----------------------- | ------------------------------------------------------- |
| CloudAdmins               | AdministratorAccess     | Accès complet à tous les services                       |
| SecurityAudit             | ReadOnlyAccess          | Accès lecture seule aux ressources et logs              |
| Indus-Dev / Fin-Dev       | DeveloperScopedPolicy   | EC2, S3, Lambda, pas de modification réseau ou sécurité |
| Indus-Ops / HR-AppSupport | OperationalAccessPolicy | Gestion opérationnelle limitée (prod, support)          |
| Fin-Compliance            | ComplianceAuditPolicy   | Lecture des données financières et audit                |

Les policies personnalisées peuvent combiner : EC2, S3, Lambda, CloudWatch et IAM en lecture seule.

## Utilisateurs Types et Attribution

| Nom   | Rôle / Groupe    |
| ----- | ---------------- |
| Jean  | CloudAdmins      |
| Amina | SecurityAudit    |
| Lucas | Indus-Dev        |
| Chloé | Fin-DataAnalysts |

## Organizational Units (OU)

| Nom de l’OU  |
| ------------ |
| OU_IT        |
| OU_Industrie |
| OU_Finance   |
| OU_RH        |

## Déploiement

### Création des OU dans AWS Organizations

#### Créer l’organisation

```Bash
aws organizations create-organization --feature-set ALL
```

Cette commande crée une nouvelle organisation AWS avec toutes les fonctionnalités activées.

- Initialise une organisation dans ton compte AWS.
- Le compte exécutant la commande devient le **compte de management (maître)**.
- Feature set `ALL` : Active toutes les fonctionnalités d’AWS Organizations, incluant :
- **Gestion complète (Full Management Features)** : création d’OU, Service Control Policies (SCP), partage de ressources.
- **Consolidated Billing** : regroupement de la facturation de tous les comptes.

#### Résultat

- Une **racine (`r-xxxx`)** est créée automatiquement.
- Tu peux ensuite créer des **OU** et ajouter des comptes AWS sous l’organisation.

#### Voir les informations sur l’organisation

```Bash
aws organizations describe-organization
```

Ce que ça retourne :

- L’ID de la racine (r-xxxx)
- L’ARN de l’organisation
- L’ID du compte de management
- Le feature set activé (ALL)

#### Lister les racines

```Bash
aws organizations list-roots
```

#### Créer les OU par secteur

```Bash
aws organizations create-organizational-unit --parent-id r-exemple --name OU_IT
aws organizations create-organizational-unit --parent-id r-exemple --name OU_Industrie
aws organizations create-organizational-unit --parent-id r-exemple --name OU_Finance
aws organizations create-organizational-unit --parent-id r-exemple --name OU_RH
```

`r-exemple` : ID de la racine de ton organisation (obtenu via `aws organizations list-roots`)

### Création des groupes IAM

```Bash
# Direction IT
aws iam create-group --group-name CloudAdmins
aws iam create-group --group-name SecurityAudit

# Secteur Industrie
aws iam create-group --group-name Indus-Dev
aws iam create-group --group-name Indus-Ops

# Secteur Finance
aws iam create-group --group-name Fin-DataAnalysts
aws iam create-group --group-name Fin-Compliance

# Ressources Humaines
aws iam create-group --group-name HR-AppSupport
```

### Création des fichiers

1. Créer le fichier `DeveloperScopedPolicy.json`

**But** :

- C’est une policy IAM personnalisée (custom policy) pour les développeurs.
- Elle définit exactement ce qu’un développeur peut faire dans AWS.

```Bash
nano DeveloperScopedPolicy.json
```

Contenu :

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:*", "s3:*", "lambda:*"],
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": ["ec2:DeleteVpc", "iam:*", "organizations:*"],
      "Resource": "*"
    }
  ]
}
```

Sauvegarde et ferme le fichier.

#### Explication :

- Allow : donne accès complet à EC2, S3 et Lambda.
- Deny : interdit la suppression des VPC et toute action IAM/Organizations (sécurité).
- Résultat : le développeur peut créer, modifier et gérer ses ressources, mais ne peut pas toucher à la sécurité ni à l’organisation.

2. Créer le fichier `OperationalAccessPolicy.json`

**But** :

- C’est une policy pour les opérations ou support, souvent pour les équipes de maintenance.
- Elle donne des droits limités en lecture ou actions spécifiques, pour éviter de casser l’infrastructure.

```Bash
nano OperationalAccessPolicy.json
```

Contenu :

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:Describe*", "s3:Get*", "cloudwatch:*"],
      "Resource": "*"
    }
  ]
}
```

#### Explication :

- ec2:Describe\* : permet de lister les instances EC2, mais pas de les modifier.
- s3:Get\* : permet de lire les données S3.
- cloudwatch: : accès complet aux métriques et logs pour monitorer l’infrastructure.
- Résultat : les utilisateurs peuvent superviser et lire, mais ne peuvent pas créer ni détruire de ressources.

3. Vérifier que les fichiers existent :

```Bash
ls -l DeveloperScopedPolicy.json OperationalAccessPolicy.json
```

Tu dois voir les deux fichiers dans le répertoire courant.

### Attribution des policies aux groupes

```Bash
# Administrateurs Cloud
aws iam attach-group-policy --group-name CloudAdmins --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Audit sécurité / conformité
aws iam attach-group-policy --group-name SecurityAudit --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam attach-group-policy --group-name Fin-Compliance --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Développeurs industriels et Finance (exemple policy custom)
aws iam put-group-policy --group-name Indus-Dev --policy-name DeveloperScopedPolicy --policy-document file://DeveloperScopedPolicy.json
aws iam put-group-policy --group-name Indus-Ops --policy-name OperationalAccessPolicy --policy-document file://OperationalAccessPolicy.json
aws iam put-group-policy --group-name Fin-DataAnalysts --policy-name DeveloperScopedPolicy --policy-document file://DeveloperScopedPolicy.json

# Support RH
aws iam put-group-policy --group-name HR-AppSupport --policy-name OperationalAccessPolicy --policy-document file://OperationalAccessPolicy.json
```

### Création des utilisateurs types et attribution aux groupes

```Bash
# IT
aws iam create-user --user-name Jean --tags Key=Role,Value=CloudArchitect
aws iam add-user-to-group --user-name Jean --group-name CloudAdmins

aws iam create-user --user-name Amina --tags Key=Role,Value=SecurityEngineer
aws iam add-user-to-group --user-name Amina --group-name SecurityAudit
```

```Bash
# Industrie
aws iam create-user --user-name Lucas --tags Key=Role,Value=Developer
aws iam add-user-to-group --user-name Lucas --group-name Indus-Dev
```

```Bash
# Finance
aws iam create-user --user-name Chloe --tags Key=Role,Value=DataAnalyst
aws iam add-user-to-group --user-name Chloe --group-name Fin-DataAnalysts
```

### Créer une politique de mot de passe

```Bash
aws iam update-account-password-policy \
  --minimum-password-length 12 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password
```

### Création des accès console et clés API

```Bash
#  Jean
aws iam create-login-profile --user-name Jean --password "StartupSecure2026!" --password-reset-required
aws iam create-access-key --user-name Jean

```

```Bash
# Amina
aws iam create-login-profile --user-name Amina --password "StartupSecure2026!" --password-reset-required
aws iam create-access-key --user-name Amina
```

```Bash
# Lucas
aws iam create-login-profile --user-name Lucas --password "StartupSecure2026!" --password-reset-required
aws iam create-access-key --user-name Lucas
```

```Bash
# Chloe
aws iam create-login-profile --user-name Chloe --password "StartupSecure2026!" --password-reset-required
aws iam create-access-key --user-name Chloe
```

### MFA pour les utilisateurs sensibles

```Bash
# Jean (Cloud Architect)
aws iam create-virtual-mfa-device --virtual-mfa-device-name JeanMFA --outfile ./JeanMFA_QR.png --bootstrap-method QRCodePNG

# Amina (Security Engineer)
aws iam create-virtual-mfa-device --virtual-mfa-device-name AminaMFA --outfile ./AminaMFA_QR.png --bootstrap-method QRCodePNG

```

Scanner et intégrer les deux images dans l'application Authenticator (Amina - Jean).

### Liste les dispositifs MFA virtuels existants et trouve celui d’Amina et de Jean

```Bash
# Amina
aws iam list-virtual-mfa-devices | jq -r '.VirtualMFADevices[] | select(.User.UserName=="Amina") | .SerialNumber'

# Jean
aws iam list-virtual-mfa-devices | jq -r '.VirtualMFADevices[] | select(.User.UserName=="Jean") | .SerialNumber'
```

### Activation du MFA pour l’utilisateur Amina & Jean

```Bash
# Amina
aws iam enable-mfa-device \
  --user-name Amina \
  --serial-number arn:aws:iam::1571748864:mfa/AminaMFA \
  --authentication-code1 116618 \
  --authentication-code2 963095
```

```Bash
# Jean
aws iam enable-mfa-device \
  --user-name Jean \
  --serial-number arn:aws:iam::1571748864:mfa/JeanMFA \
  --authentication-code1 773391 \
  --authentication-code2 147160
```

> Remplacer le code1 et le code2 par vos vrais codes (Amina & Jean)
> Code1 = ..........
> Code2 = ...........

#### Après scan, supprimer les images QR pour sécurité

```Bash
rm ./AminaMFA_QR.png
rm ./JeanMFA_QR.png
```

### Gestion des politiques

Ces fichiers JSON sont appelés “trust policies” ou politiques d’approbation. Ils définissent qui ou quoi est autorisé à assumer un rôle IAM dans AWS.

1. EC2 : Crée un fichier `trust-policy-ec2.json` avec ce contenu

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

2. Lambda : Crée un fichier `trust-policy-lambda.json` avec ce contenu

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Création des rôles IAM pour services

```Bash
# Rôle EC2 pour accès S3 complet
aws iam create-role --role-name EC2S3AccessRole --assume-role-policy-document file://trust-policy-ec2.json
aws iam attach-role-policy --role-name EC2S3AccessRole --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

```

```Bash
# Rôle Lambda pour accès aux logs CloudWatch
aws iam create-role --role-name LambdaLogsRole --assume-role-policy-document file://trust-policy-lambda.json
aws iam attach-role-policy --role-name LambdaLogsRole --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```

### Nettoyage

1. Créer le fichier `clean.sh`

```Bash
nano clean.sh
```

2. Colle le contenu suivant dans `clean.sh` :

```bash
#!/bin/bash
# Supprime utilisateurs, MFA, clés API, groupes, policies, rôles et OU

# --- Configuration des utilisateurs du lab ---
users=("Jean" "Amina" "Lucas" "Chloe")

echo "============================"
echo "Suppression des utilisateurs IAM"
echo "============================"

for user in "${users[@]}"; do
  echo "Traitement de l'utilisateur : $user"

  # Supprimer toutes les Access Keys
  aws iam list-access-keys --user-name "$user" | jq -r '.AccessKeyMetadata[].AccessKeyId' | while read key; do
    echo "Suppression de la clé : $key"
    aws iam delete-access-key --user-name "$user" --access-key-id "$key"
  done

  # Supprimer les MFA (s'il y en a)
  aws iam list-virtual-mfa-devices | jq -r ".VirtualMFADevices[] | select(.User.UserName==\"$user\") | .SerialNumber" | while read serial; do
    echo "Désactivation MFA : $serial"
    aws iam deactivate-mfa-device --user-name "$user" --serial-number "$serial"
    echo "Suppression MFA virtuelle : $serial"
    aws iam delete-virtual-mfa-device --serial-number "$serial"
  done

  # Retirer l'utilisateur de tous les groupes
  aws iam list-groups-for-user --user-name "$user" | jq -r '.Groups[].GroupName' | while read group; do
    echo "Retrait de $user du groupe $group"
    aws iam remove-user-from-group --user-name "$user" --group-name "$group"
  done

  # Supprimer le profil de connexion console
  echo "Suppression du login console"
  aws iam delete-login-profile --user-name "$user" 2>/dev/null

  # Supprimer l'utilisateur
  echo "Suppression de l'utilisateur IAM"
  aws iam delete-user --user-name "$user"
done

echo "============================"
echo "Détachement et suppression des policies des groupes"
echo "============================"

aws iam detach-group-policy --group-name CloudAdmins --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam detach-group-policy --group-name SecurityAudit --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam detach-group-policy --group-name Fin-Compliance --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

aws iam delete-group-policy --group-name Indus-Dev --policy-name DeveloperScopedPolicy
aws iam delete-group-policy --group-name Indus-Ops --policy-name OperationalAccessPolicy
aws iam delete-group-policy --group-name Fin-DataAnalysts --policy-name DeveloperScopedPolicy
aws iam delete-group-policy --group-name HR-AppSupport --policy-name OperationalAccessPolicy

echo "============================"
echo "Suppression des groupes IAM"
echo "============================"

aws iam delete-group --group-name CloudAdmins
aws iam delete-group --group-name SecurityAudit
aws iam delete-group --group-name Indus-Dev
aws iam delete-group --group-name Indus-Ops
aws iam delete-group --group-name Fin-DataAnalysts
aws iam delete-group --group-name Fin-Compliance
aws iam delete-group --group-name HR-AppSupport

echo "============================"
echo "Détachement et suppression des rôles IAM"
echo "============================"

aws iam detach-role-policy --role-name EC2S3AccessRole --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam detach-role-policy --role-name LambdaLogsRole --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

aws iam delete-role --role-name EC2S3AccessRole
aws iam delete-role --role-name LambdaLogsRole

echo "============================"
echo "Suppression des OU AWS Organizations"
echo "============================"

# Récupération automatique des IDs des OU
ou_ids=$(aws organizations list-organizational-units-for-parent --parent-id $(aws organizations list-roots | jq -r '.Roots[0].Id') | jq -r '.OrganizationalUnits[].Id')

for ou_id in $ou_ids; do
  ou_name=$(aws organizations describe-organizational-unit --organizational-unit-id "$ou_id" | jq -r '.OrganizationalUnit.Name')
  echo "Suppression de l'OU $ou_name ($ou_id)"

  # Vérifie si l'OU contient des comptes
  accounts=$(aws organizations list-accounts-for-parent --parent-id "$ou_id" | jq -r '.Accounts[].Id')
  for acct in $accounts; do
    echo "Déplacer ou supprimer le compte $acct avant de supprimer l'OU"
  done

  # Supprimer l'OU si vide
  aws organizations delete-organizational-unit --organizational-unit-id "$ou_id" 2>/dev/null || echo "Impossible de supprimer $ou_name, OU non vide ou permissions insuffisantes"
done

echo "Toutes les ressources IAM et OU du lab ont été supprimées."
```

3. Rendre le fichier exécutable

```Bash
chmod +x ./clean.sh
```

4. Exécuter le script

```Bash
./clean.sh
```
