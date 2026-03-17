# LAB : IAM (user - Group - Policy)

## Objectif du Lab

L’objectif de ce laboratoire est de **maîtriser la gestion des identités et des accès (IAM) sur AWS**,  
afin de sécuriser les ressources cloud et de préparer un environnement DevOps sécurisé.

### À la fin de ce lab, tu seras capable de :

- Créer des utilisateurs AWS et gérer leurs permissions
- Configurer des groupes avec des politiques prédéfinies (ex. `AdministratorAccess`)
- Gérer l’accès via console et API (CLI) avec les clés d’accès
- Activer et configurer le **MFA (Multi-Factor Authentication)** pour sécuriser les comptes
- Configurer le profil AWS CLI pour automatiser les tâches DevOps
- Supprimer correctement les utilisateurs et leurs accès, en respectant les bonnes pratiques de sécurité

---

## Prérequis

Avant de démarrer ce lab, tu dois disposer de :

- Un compte AWS avec droits administrateur
- AWS CLI installé et configuré sur ton poste
- Connaissance de base de la ligne de commande
- Outil de lecture QR Code pour activer le MFA
- PowerShell ou terminal compatible

## IAM and Security (Identity and Access Management)

### Utilisateur / Groupes

#### 1. Création de l'utilisateur

```PowerShell
aws iam create-user --user-name DevOpsUser --tags Key=Role,Value=DevOps
```

![image](https://hackmd.io/_uploads/BJi-d1kcbg.png)
![image](https://hackmd.io/_uploads/BJQrOJJ9Zg.png)

#### 2. Création de l'accès Console (Mot de passe)

```PowerShell
aws iam create-login-profile --user-name DevOpsUser --password "StartupSecure2026!" --password-reset-required
```

![image](https://hackmd.io/_uploads/BJH__JyqWe.png)

#### 3. Génération des clés d'accès API (CLI)

```PowerShell
aws iam create-access-key --user-name DevOpsUser
```

![image](https://hackmd.io/_uploads/B1g0n_kJcZg.png)

#### 4. Création du groupe

```PowerShell
aws iam create-group --group-name Group-Admins
```

![image](https://hackmd.io/_uploads/ryZNKJycZx.png)
![image](https://hackmd.io/_uploads/ByxDBKykc-e.png)

#### 5. Attribution des droits Admin au groupe

```Powershell
aws iam attach-group-policy --group-name Group-Admins --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

![image](https://hackmd.io/_uploads/HkQcKyk5-l.png)

![image](https://hackmd.io/_uploads/BJsjYJ1c-l.png)

#### 6. Ajouter l'utlisateur au groupe

```PowerShell
aws iam add-user-to-group --user-name DevOpsUser --group-name Group-Admins
```

![image](https://hackmd.io/_uploads/BkOTK1JqZe.png)
![image](https://hackmd.io/_uploads/rJF1ckJcWg.png)

#### 7. Préparation du MFA (Génère le QR Code pour l'utilisateur)

```PowerShell
aws iam create-virtual-mfa-device --virtual-mfa-device-name DevOpsUserMFA --outfile ./MFA_QRCode.png --bootstrap-method QRCodePNG
echo "L'utilisateur DevOpsUser a été créé. Scannez MFA_QRCode.png pour finaliser le MFA."
```

![image](https://hackmd.io/_uploads/BkpNsyJ5Zl.png)

#### 8. Supprimer l'image après scan

```PowerShell
Remove-Item ./MFA_QRCode.png
```

![image](https://hackmd.io/_uploads/rJajaky5-x.png)

## Configure CLI

### Lister les clés existantes

```PowerShell
aws iam list-access-keys --user-name DevOpsUser
```

![image](https://hackmd.io/_uploads/rymuWgyqZg.png)

### Configurer le profil

```PowerShell
$ aws configure
AWS Access Key ID [None]: AKIAEXAMPLE
AWS Secret Access Key [None]: abc123EXAMPLEKEY
Default region name [None]: eu-west-3
Default output format [None]: json
```

### vérifier que ta configuration CLI fonctionne bien

```PowerShell
aws sts get-caller-identity
```

## Supression

### Il faut d'abord lister l'AccessKeyId :

```
aws iam list-access-keys --user-name DevOpsUser
```

![image](https://hackmd.io/_uploads/rkARAWycbg.png)

#### Supprimer les clés d'accès de l'utlisateur

```PowerShell
aws iam delete-access-key --user-name DevOpsUser --access-key-id AKIASJFLG55HW5FPS3CJ
```

![image](https://hackmd.io/_uploads/HyO81GycWg.png)

### Retirer l'utilisateur du groupe

```PowerShell
aws iam remove-user-from-group --group-name Group-Admins --user-name DevOpsUser
```

![image](https://hackmd.io/_uploads/r10PJz1c-g.png)

![image](https://hackmd.io/_uploads/H1t5kfJqWx.png)

### Supprimer le profil de connexion (mot de passe console)

```PowerShell
aws iam delete-login-profile --user-name DevOpsUser
```

![image](https://hackmd.io/_uploads/ryYnyzJ5Ze.png)

![image](https://hackmd.io/_uploads/r1h0JGyqZe.png)

### Enfin, supprimer l'utilisateur

```PowerShell
aws iam delete-user --user-name DevOpsUser
```

![image](https://hackmd.io/_uploads/rkXxgzyqZx.png)

![image](https://hackmd.io/_uploads/HkQfeGJqZl.png)
