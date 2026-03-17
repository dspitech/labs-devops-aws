# LAB : Créer une Alerte de Budget AWS avec PowerShell

Ce lab montre comment créer un **budget AWS avec notification par email** pour surveiller les coûts.

## Remarque

Ce lab doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

---

## Étape 1 : Créer le fichier de configuration du budget

Crée un fichier nommé `budget.json` dans ton dossier de projet avec ce contenu :

```json
{
  "BudgetName": "Alerte-Zero-Dollar",
  "BudgetLimit": {
    "Amount": "1.00",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
```

Ici, le budget est fixé à 1 $ par mois pour déclencher une alerte dès qu’on dépasse 80 % de ce montant.

## Étape 2 : Créer le fichier de notification

Crée un fichier nommé `notifications.json` et remplace `ton-email@exemple.com` par ton adresse email réelle :

```json
[
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      {
        "SubscriptionType": "EMAIL",
        "Address": "ton-email@exemple.com"
      }
    ]
  }
]
```

Cette notification sera envoyée lorsque les coûts réels atteindront 80 % du budget.

## Étape 3 : Exécuter la création du budget

Ouvre PowerShell dans le dossier où se trouvent tes fichiers `budget.json` et `notifications.json`, puis lance :

```PowerShell
# Recupere ton ID de compte AWS automatiquement
$AccountId = (aws sts get-caller-identity --query "Account" --output text)

# Cree le budget
aws budgets create-budget `
    --account-id $AccountId `
    --budget file://budget.json `
    --notifications-with-subscribers file://notifications.json
```

Après exécution, ton budget avec alerte par email est créé.

## Vérifier que le budget est actif

Pour lister tous les budgets existants sur ton compte :

```PowerShell
aws budgets describe-budgets --account-id $AccountId --query "Budgets[*].BudgetName" --output table
```

Tu devrais voir `Alerte-Zero-Dollar` dans la liste.

## Astuce

- Les notifications par email nécessitent une confirmation : tu recevras un email AWS pour valider l’abonnement.
- Tu peux ajuster Threshold dans notifications.json pour recevoir des alertes à différents pourcentages (ex. 50, 90, 100)
