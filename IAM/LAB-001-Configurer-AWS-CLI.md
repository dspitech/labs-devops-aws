# Lab : Préparer un utilisateur IAM et configurer AWS CLI

## Objectif

- Créer un groupe
- Attacher la politique `AdministratorAccess`
- Créer un utilisateur
- Ajouter l’utilisateur au groupe
- Générer une clé d’accès
- Configurer AWS CLI avec la clé

---

## Prérequis

- Compte AWS avec droits d’administrateur
- Terminal (CloudShell ou local)
- Connaissances de base sur IAM, PowerShell et AWS CLI

---

## Se connecter à la console AWS

![image](https://hackmd.io/_uploads/H1e9gjL9be.png)

## Choisir le service IAM

![image](https://hackmd.io/_uploads/HkUnxsIqZx.png)

## Créer un groupe IAM avec droits administrateur

![image](https://hackmd.io/_uploads/Hytf-iIqbx.png)

- Nom du groupe : `AdminUsers`

![image](https://hackmd.io/_uploads/rkasWi89Zx.png)

- Rôle : AdministratorAccess
  ![image](https://hackmd.io/_uploads/Hy_yzs85Ze.png)

![image](https://hackmd.io/_uploads/HkW7zoUqWe.png)

## Créer un utilisateur et l'ajouter dans ce groupe

![image](https://hackmd.io/_uploads/BJyIGoU5Ze.png)

- Nom : `UserAdmin`
- Cocher : Fournir aux utilisateurs l'accès à la console de gestion AWS
  ![image](https://hackmd.io/_uploads/SygcMiLqZe.png)

- Mot de passe temporaire : `Password1234`
- Cocher la case : Les utilisateurs doivent créer un nouveau mot de passe lors de la prochaine connexion
  ![image](https://hackmd.io/_uploads/B13RMiLqWg.png)
- Ajouter cet utilisateur dans le groupe `AdminUsers`
  ![image](https://hackmd.io/_uploads/ByUQmiIqWl.png)
- Détails de l'utilisateur
  ![image](https://hackmd.io/_uploads/HJm_mi8cZl.png)
  ![image](https://hackmd.io/_uploads/BJSF7i85bx.png)
  ![image](https://hackmd.io/_uploads/r1VsXi89-e.png)

## Créer une clé d'accès

- Aller dans IAM
- Dans personnes
  ![image](https://hackmd.io/_uploads/B1-wNsL9Zx.png)
- Sélectionner l'utilisateur concerné
  ![image](https://hackmd.io/_uploads/BJaOVsLc-l.png)
- Aller dans le champs : Informations d'identification de sécurité
  ![image](https://hackmd.io/_uploads/BkNqEs8cbg.png)
- En bash cliquer sur le bouton : créer une clé d'accès > remplir les infos
  ![image](https://hackmd.io/_uploads/SJKoNs8qWx.png)
- Choisir : Interface de ligne de commande (CLI)
  ![image](https://hackmd.io/_uploads/BkQBSjI9Wg.png)
- En bas cocher la case : Je comprends la recommandation ci-dessus et je souhaite procéder à la création d'une clé d'accès.
  ![image](https://hackmd.io/_uploads/SysDHsIqWx.png)
- Cliquer sur suivant
  ![image](https://hackmd.io/_uploads/HyItriLcbe.png)
- Définir une description de la valeur d'identification
  ![image](https://hackmd.io/_uploads/rJiZ8iIcbe.png)
- Cliquer sur le bouton : créer une clé d'accès
  ![image](https://hackmd.io/_uploads/Hkf7Lj85bl.png)
- Copier les infos de la clé (Clé d'accès et Clé d'accès secrète) ou télécharger le fichier CSV
  ![image](https://hackmd.io/_uploads/BkFUIoL9bg.png)
  La clé d'accès à la consle pour cet utilisateur a été créé avec succès.![image](https://hackmd.io/_uploads/HyWTLjL5Wl.png)

## Configurer AWS CLI avec cet utilisateur

- Lancer le cloud shell
  ![image](https://hackmd.io/_uploads/Sy4ZPiL5Zg.png)
  ![image](https://hackmd.io/_uploads/SyVNviIqWe.png)
- Vérifier la version
  ![image](https://hackmd.io/_uploads/S1Iwwj8qWl.png)
- Configurer l'accès

```Bash
aws configure
```

![image](https://hackmd.io/_uploads/B1sb_jI5Zg.png)

- Entrer Access Key ID de l'utilisateur
  ![image](https://hackmd.io/_uploads/H1Y7uo89be.png)
- Entrer le secret Access Key de l'utilisateur
  ![image](https://hackmd.io/_uploads/HyEHds89-l.png)
- Choisir la Default region (ex: eu-west-3)
  ![image](https://hackmd.io/_uploads/BJlv_sI9We.png)
- Choisir le format par défaut (json ou table)
  ![image](https://hackmd.io/_uploads/HkK_OsU5We.png)
- Configuration terminée
  ![image](https://hackmd.io/_uploads/SJg9_jU9Wl.png)

## Vérifier la configuration

```Bash
aws sts get-caller-identity
```

![image](https://hackmd.io/_uploads/r1p1Yj89bl.png)

## Lister les groupes

```Bash
aws iam list-groups --query "Groups[*].GroupName" --output table
```

![image](https://hackmd.io/_uploads/rJmrFsUqbg.png)

## Configurer la CLI dans PowerShell

### Télécharger et nstaller la CLI

Lien : [install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

### Vérifier la version

```bash
aws --version
```

![image](https://hackmd.io/_uploads/SJphqiLcZe.png)

### Configurer l'accès

```Bash
aws configure
```

![image](https://hackmd.io/_uploads/SyMgss8c-g.png)

- Entrer Access Key ID de l'utilisateur
  ![image](https://hackmd.io/_uploads/B1QfjiI9bx.png)
- Entrer le secret Access Key de l'utilisateur
  ![image](https://hackmd.io/_uploads/SySQooIq-l.png)
- Choisir la Default region (ex: eu-west-3)
  ![image](https://hackmd.io/_uploads/BJPVssIq-e.png)
- Choisir le format par défaut (json ou table)
  ![image](https://hackmd.io/_uploads/r1vrss8q-g.png)
- Configuration terminée
  ![image](https://hackmd.io/_uploads/BJEIojIcWx.png)

### Vérifier la configuration

```
aws sts get-caller-identity
```

![image](https://hackmd.io/_uploads/BJIoij89be.png)

## Lister les groupes

```Bash
aws iam list-groups --query "Groups[*].GroupName" --output table
```

![image](https://hackmd.io/_uploads/B1MTijIcbx.png)
