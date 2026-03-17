# LAB AWS DynamoDB : Basics and Tables

## Objectif

- Créer une **table DynamoDB**
- **Ajouter**, **lire**, **mettre à jour** et **supprimer** des items
- Créer un **index secondaire global (GSI)** et faire des **requêtes filtrées**

---

## DynamoDB

**DynamoDB** est une base de données **NoSQL managée** par AWS, conçue pour des performances rapides et prévisibles à n’importe quelle échelle. Elle est **serverless**, donc AWS gère le provisionnement des serveurs, le scaling et la maintenance.

### Concepts clés

- **Table** : Conteneur principal pour les items (équivalent à une table dans une base relationnelle)
- **Item** : Une entrée dans la table (équivalent à une ligne)
- **Attribute** : Une donnée dans un item (équivalent à une colonne)
- **Primary Key** : Clé unique qui identifie chaque item. Peut être :
  - **Partition Key** : clé simple
  - **Partition Key + Sort Key** : clé composite pour ordonner les items
- **Index secondaire global (GSI)** : Permet de faire des requêtes avec une clé différente de la clé primaire
- **Provisioned / On-Demand Capacity** : Choix entre un débit fixe ou évolutif automatiquement
- **Streams** : Flux de modifications pour déclencher des actions (ex. via Lambda)

### Avantages

- Haute performance à grande échelle
- Haute disponibilité et durabilité (réplication multi-AZ)
- Flexible pour stocker des données structurées ou semi-structurées
- Intégration facile avec AWS Lambda, API Gateway, et d’autres services AWS

### Bonnes pratiques

- Choisir des clés partition efficaces pour éviter les **hot partitions**
- Utiliser des GSI pour les requêtes fréquentes mais non basées sur la clé primaire
- Activer les **streams** pour auditer ou synchroniser les données

## Remarque

Ce lab doit être réalisé dans **AWS CloudShell**.

## Déploiement

### Créer la table DevOpsUsers

```Bash
aws dynamodb create-table \
    --table-name DevOpsUsers \
    --attribute-definitions AttributeName=UserId,AttributeType=S \
    --key-schema AttributeName=UserId,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/ry9lVnZc-x.png)
![image](https://hackmd.io/_uploads/BkwME3W5Wl.png)

#### Explication des options pour `create-table`

- `--table-name DevOpsUsers`  
  Nom de la **table** à créer.
- `--attribute-definitions AttributeName=UserId,AttributeType=S`  
  Définition des **attributs utilisés dans la clé primaire**.  
  Ici, `UserId` de type `S` (**String**).
- `--key-schema AttributeName=UserId,KeyType=HASH`  
  Définition de la **clé primaire** : `UserId` est la **Partition Key (HASH)**.
- `--provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5`  
  **Capacité provisionnée** : 5 lectures et 5 écritures par seconde.
- `--region eu-west-3`  
  **Région AWS** où la table sera créée (ici **Paris**).

### Vérifier que la table a bien été créée.

```Bash
aws dynamodb describe-table --table-name DevOpsUsers --region eu-west-3
```

![image](https://hackmd.io/_uploads/SJg_E2W5bl.png)

### Ajouter un utilisateur

```Bash
aws dynamodb put-item \
    --table-name DevOpsUsers \
    --item '{
        "UserId": {"S": "U1001"},
        "Name": {"S": "Sainath"},
        "Role": {"S": "DevOps"}
    }' \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/HJAHN3b5be.png)
![image](https://hackmd.io/_uploads/HkViE3bq-e.png)
![image](https://hackmd.io/_uploads/r1Xn4nW9-g.png)
![image](https://hackmd.io/_uploads/rkeRV3b5Wl.png)

### Lire un utilisateur par UserId

```Bash
aws dynamodb get-item \
    --table-name DevOpsUsers \
    --key '{"UserId": {"S": "U1001"}}' \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/Hy1xShZqZx.png)

### Mettre à jour le rôle d’un utilisateur

```Bash
aws dynamodb update-item \
    --table-name DevOpsUsers \
    --key '{"UserId": {"S": "U1001"}}' \
    --update-expression "SET #R = :r" \
    --expression-attribute-names '{"#R": "Role"}' \
    --expression-attribute-values '{":r": {"S": "Senior DevOps"}}' \
    --region eu-west-3
```

#### Explications des éléments de l'update-expression

- `#R` : **alias** que tu donnes à l’attribut `Role`
- `"SET #R = :r"` : indique que tu veux **mettre à jour l’attribut `Role`** avec la valeur `:r`
- `:r` : **valeur à mettre**, ici `"Senior DevOps"`
- `--expression-attribute-names` : dictionnaire qui **associe `#R` à l’attribut `Role`**

![image](https://hackmd.io/_uploads/Hkc1Uhb5We.png)
![image](https://hackmd.io/_uploads/HkQPLhZ9-g.png)

### Vérifier la mise à jour

```Bash
aws dynamodb get-item \
    --table-name DevOpsUsers \
    --key '{"UserId": {"S": "U1001"}}' \
    --region eu-west-3
```

### Supprimer un utilisateur

```Bash
aws dynamodb delete-item \
    --table-name DevOpsUsers \
    --key '{"UserId": {"S": "U1001"}}' \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/BJ0Y8nbq-e.png)

## DynamoDB : Clé primaire vs Index

Chaque table DynamoDB a une **clé primaire** qui identifie de manière unique chaque élément (row).

### Exemple

**Table : DevOpsUsers**

| UserId (clé primaire) | Name    | Role     |
| --------------------- | ------- | -------- |
| U1001                 | Sainath | DevOps   |
| U1002                 | Alice   | SysAdmin |

Ici : **UserId** = clé primaire (Partition Key).  
Tous les items doivent avoir un **UserId unique**.

---

### 🔹 Pourquoi un GSI ?

Parfois, tu veux faire des **recherches rapides** sur un attribut qui n’est **pas la clé primaire**.

Exemple : retrouver tous les DevOps sans connaître leur UserId.

**Problème :** DynamoDB ne peut pas filtrer rapidement par **Role** si ce n’est pas la clé primaire → ce serait lent.  
**Solution :** créer un **Global Secondary Index (GSI)** sur Role.

---

### 🔹 Qu’est-ce qu’un GSI ?

Un **GSI** est comme une **mini-table dérivée** qui te permet :

- D’indexer un ou plusieurs attributs différents de la clé primaire
- De faire des **requêtes rapides** sur ces attributs
- D’avoir sa propre **clé de partition** (Partition Key) et **clé de tri** (optionnelle)

### Exemple : GSI RoleIndex

| Role (PK) | UserId (Sort Key) |
| --------- | ----------------- |
| DevOps    | U1001             |
| DevOps    | U1003             |
| SysAdmin  | U1002             |

### Différence avec une Local Secondary Index (LSI)

- **LSI (Local Secondary Index)** :
  - Index sur un **autre attribut**
  - **Doit partager la même clé de partition** que la table principale

- **GSI (Global Secondary Index)** :
  - Index **totalement indépendant**
  - Tu peux choisir une **clé complètement différente**

### Ajouter un Index Secondaire Global (GSI) sur l’attribut `Role`

```Bash
aws dynamodb update-table \
    --table-name DevOpsUsers \
    --attribute-definitions AttributeName=Role,AttributeType=S \
    --global-secondary-index-updates '[
        {
            "Create": {
                "IndexName": "RoleIndex",
                "KeySchema": [{"AttributeName": "Role", "KeyType": "HASH"}],
                "Projection": {"ProjectionType": "ALL"},
                "ProvisionedThroughput": {"ReadCapacityUnits": 5, "WriteCapacityUnits": 5}
            }
        }
    ]' \
    --region eu-west-3
```

#### Explication des parties d'une commande `update-table` pour GSI

- **`aws dynamodb update-table`**  
  Commande pour **mettre à jour une table DynamoDB existante**.  
  Ici, on met à jour **DevOpsUsers** pour ajouter un **Global Secondary Index (GSI)**.
- **`--table-name DevOpsUsers`**  
  Nom de la table à modifier.
- **`--attribute-definitions AttributeName=Role,AttributeType=S`**  
  DynamoDB a besoin de savoir le **type de l’attribut utilisé dans l’index**.
- `Role` = nom de l’attribut
- `S` = type String
  > Important : même si `Role` existe déjà dans la table, il faut le définir pour l’index.
- **`--global-secondary-index-updates '[...]'`**  
  Permet de **créer ou supprimer un GSI**.  
  Ici, `"Create": { ... }` → créer un nouvel index.
- **`"IndexName": "RoleIndex"`**  
  Nom que l’on donne à l’index secondaire.  
  Cet index permet d’effectuer des **requêtes sur l’attribut `Role`**.
- **`"KeySchema": [{"AttributeName": "Role", "KeyType": "HASH"}]`**  
  Définition de la **clé de l’index**.
- `HASH` = clé principale de l’index (similaire à la Partition Key).  
  Ici, on peut interroger tous les utilisateurs selon leur `Role`.
- **`"Projection": {"ProjectionType": "ALL"}`**  
  Définit quelles colonnes seront disponibles via l’index :
- `ALL` → toutes les colonnes de la table
- `KEYS_ONLY` → seulement la clé primaire et la clé de l’index
- `INCLUDE` → seulement certaines colonnes (spécifiées dans `NonKeyAttributes`)
- **`"ProvisionedThroughput": {"ReadCapacityUnits": 5, "WriteCapacityUnits": 5}`**  
  Pour les tables **provisionnées**, il faut définir la **capacité pour l’index** :
- `ReadCapacityUnits` = nombre d’opérations de lecture par seconde
- `WriteCapacityUnits` = nombre d’opérations d’écriture par seconde

  > Si tu es en mode **On-Demand (PAY_PER_REQUEST)**, tu n’as pas besoin de cette section.

- **`--region eu-west-3`**  
  Région AWS où la table est située (ici **Paris**).

![image](https://hackmd.io/_uploads/Sytow3ZqZe.png)

![image](https://hackmd.io/_uploads/B1c6PnW9Wg.png)

### Vérifier l’état du GSI régulièrement

```Bash
aws dynamodb describe-table --table-name DevOpsUsers --region eu-west-3 \
--query "Table.GlobalSecondaryIndexes[*].{IndexName:IndexName,Status:IndexStatus}"
```

![image](https://hackmd.io/_uploads/SyeTKnZqWe.png)
![image](https://hackmd.io/_uploads/rkQAtnWcZx.png)

### Ajouter un item

```Bash
aws dynamodb put-item \
    --table-name DevOpsUsers \
    --item '{"UserId": {"S": "U1002"}, "Name": {"S": "Alice"}, "Role": {"S": "DevOps"}}' \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/rJZZjhZ9be.png)

### Requête sur le GSI pour lister les utilisateurs par rôle

```Bash
aws dynamodb update-item \
    --table-name DevOpsUsers \
    --key '{"UserId": {"S": "U1001"}}' \
    --update-expression "SET #R = :newrole" \
    --expression-attribute-names '{"#R": "Role"}' \
    --expression-attribute-values '{":newrole": {"S": "Senior DevOps"}}' \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/rJcms3b9We.png)

### Nettoyage

#### Supprimer la table DynamoDB

```Bash
aws dynamodb delete-table --table-name DevOpsUsers --region eu-west-3
```

![image](https://hackmd.io/_uploads/SkFYi3-c-x.png)

#### Vérifie que la table est supprimée

```Bash
aws dynamodb list-tables --region eu-west-3
```

![image](https://hackmd.io/_uploads/HJihonbqWx.png)
