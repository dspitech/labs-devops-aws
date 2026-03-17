# LAB : AWS S3 – Buckets, Versioning et Lifecycle Policies

## Description

Ce laboratoire montre comment :

- créer un **bucket S3**
- activer le **versioning**
- uploader des fichiers
- lister les objets
- créer et appliquer une **lifecycle policy** pour archiver des objets vers **Glacier** après 30 jours

**S3** est un service AWS de **stockage objet scalable, durable et sécurisé**.  
Ce lab est conçu pour les environnements **DevOps**.

---

## Objectifs

À la fin de ce lab, tu sauras :

- créer et gérer un **bucket S3**
- activer et comprendre le **versioning**
- transférer des fichiers vers S3 via **AWS CLI**
- créer et appliquer une **lifecycle policy**
- gérer et vérifier plusieurs versions d’objets

---

## Concepts clés

### S3 Bucket

Un **conteneur logique** pour stocker des objets dans AWS.  
Chaque bucket a un **nom unique global** et est associé à une **région**.

---

### Versioning

Permet de **conserver toutes les versions d’un objet** dans un bucket.

**Avantages :**

- restauration de versions précédentes
- protection contre les suppressions accidentelles
- suivi des changements

---

### Lifecycle Policy

Une politique qui **automatise la gestion des objets** :

- transition vers une **classe de stockage moins coûteuse**
- suppression automatique après une durée définie

**Exemple :**  
Archiver les fichiers vers **Glacier** après 30 jours pour réduire les coûts.

---

### Classes de stockage S3

| Classe              | Description                                 |
| ------------------- | ------------------------------------------- |
| STANDARD            | Haute disponibilité et faible latence       |
| INTELLIGENT_TIERING | Optimise le coût selon l’usage              |
| GLACIER             | Archivage à faible coût, récupération lente |
| DEEP_ARCHIVE        | Archivage ultra long terme                  |

## Remarque

Ce lab doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

## Déploiement

### Étape 1 : Création du bucket

```Bash
aws s3 mb s3://estiam-bucket-2026 --region eu-west-3
```

![image](https://hackmd.io/_uploads/Sy3VYSWcWg.png)

![image](https://hackmd.io/_uploads/ryCIYHb9Zx.png)

Vérifie que le bucket a été créé :

```Bash
aws s3 ls
```

![image](https://hackmd.io/_uploads/Hy6dFSWqZl.png)

### 🛡️ Étape 2 : Activer le versioning

```Bash
aws s3api put-bucket-versioning \
--bucket estiam-bucket-2026 \
--versioning-configuration Status=Enabled
```

![image](https://hackmd.io/_uploads/HytjYSW5bl.png)

![image](https://hackmd.io/_uploads/Sk7L5BZqZg.png)

Vérifie que le versioning est activé :

```Bash
aws s3api get-bucket-versioning --bucket my-devops-bucket
```

![image](https://hackmd.io/_uploads/H1M15rW5Wl.png)

![image](https://hackmd.io/_uploads/ByCScrWcbg.png)

### Étape 3 : Créer un petit fichier de test

```Bash
echo "Test S3 Free Tier" > test.txt
cat test.txt
```

![image](https://hackmd.io/_uploads/S1R_9HW5Wl.png)

![image](https://hackmd.io/_uploads/HysncHZcbg.png)

### 🖥️ Étape 4 : Upload du fichier dans le bucket

```Bash
aws s3 cp test.txt s3://estiam-bucket-2026/
```

![image](https://hackmd.io/_uploads/Hy-AqH-cbx.png)

![image](https://hackmd.io/_uploads/B1v-oSWqWl.png)

![image](https://hackmd.io/_uploads/rya8sSWqZl.png)

Vérifie le contenu du bucket :

```Bash
aws s3 ls s3://estiam-bucket-2026/
```

![image](https://hackmd.io/_uploads/Skv_sSb9Wg.png)

### 🔄 Étape 5 : Créer une Lifecycle Policy pour archiver vers Glacier

#### 1️⃣ Crée un fichier JSON lifecycle.json

```Bash
cat <<EOF > lifecycle.json
{
  "Rules": [
    {
      "ID": "TestLifecycle",
      "Status": "Enabled",
      "Prefix": "",
      "Transitions": [
        {
          "Days": 365,
          "StorageClass": "GLACIER"
        }
      ]
    }
  ]
}
EOF
```

![image](https://hackmd.io/_uploads/SkucoB-5-l.png)

#### 2️⃣ Applique la policy

```Bash
aws s3api put-bucket-lifecycle-configuration \
--bucket estiam-bucket-2026 \
--lifecycle-configuration file://lifecycle.json
```

![image](https://hackmd.io/_uploads/rJA73rZc-e.png)

![image](https://hackmd.io/_uploads/BJCrhB-qWg.png)
![image](https://hackmd.io/_uploads/rkCwhSbqWg.png)

#### 3️⃣ Vérifie la policy appliquée

```Bash
aws s3api get-bucket-lifecycle-configuration \
--bucket estiam-bucket-2026
```

![image](https://hackmd.io/_uploads/SkC53S-9Zg.png)

⚠️ Les fichiers resteront dans STANDARD pendant au moins 365 jours → aucun coût.

#### 🧪 Étape 6 : Tester le versioning

##### 1️⃣ Modifie le fichier test.txt

```Bash
echo "Nouvelle version 2 du fichier" >> test.txt
```

![image](https://hackmd.io/_uploads/SyJJTSZcZx.png)
![image](https://hackmd.io/_uploads/H11M6HZ9-e.png)

#### 2️⃣ Re-upload

```Bash
aws s3 cp test.txt s3://estiam-bucket-2026/
```

![image](https://hackmd.io/_uploads/HJg4TBbcbe.png)

#### 3️⃣ Liste les versions

```Bash
aws s3api list-object-versions --bucket estiam-bucket-2026 --prefix test.txt
```

![image](https://hackmd.io/_uploads/BJwcaH-cbg.png)

![image](https://hackmd.io/_uploads/r1d2arWcZg.png)

![image](https://hackmd.io/_uploads/rkx1CBZcbe.png)

![image](https://hackmd.io/_uploads/BkcvRSbcWx.png)

### 🧹 Étape 7 : Nettoyage

#### Supprimer toutes les versions

```Bash
aws s3api list-object-versions --bucket estiam-bucket-2026 --query 'Versions[].{Key:Key,VersionId:VersionId}' --output text | \
while read key version; do
  aws s3api delete-object --bucket estiam-bucket-2026 --key "$key" --version-id "$version"
done
```

![image](https://hackmd.io/_uploads/rk2FAHb5Zx.png)

![image](https://hackmd.io/_uploads/SJaiCHW9-l.png)

#### Supprimer le bucket

```Bash
aws s3 rb s3://estiam-bucket-2026
```

![image](https://hackmd.io/_uploads/HJnp0rbcWx.png)

![image](https://hackmd.io/_uploads/SkElyUb9Zx.png)
