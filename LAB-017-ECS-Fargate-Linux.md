# LAB : Déploiement d’une Tâche Amazon ECS Linux pour Fargate

## Objectif du lab

Ce lab vous guide pour déployer un **service web Apache dans un conteneur Linux sur Amazon ECS avec Fargate**.  
À la fin, vous aurez :

- Un cluster ECS fonctionnel.
- Une tâche Fargate qui exécute un conteneur Apache.
- Un service ECS qui maintient la tâche active.
- Un accès web via IP publique pour tester votre application.
- Une procédure complète de nettoyage pour éviter les coûts.

---

## Description

Amazon ECS (Elastic Container Service) permet de gérer des conteneurs Docker de manière scalable.  
Fargate est le mode serverless : vous n’avez pas besoin de gérer des instances EC2, AWS se charge de la mise en place et de la maintenance.

Le lab couvre :

1. Création du cluster ECS (Fargate).
2. Définition d’une tâche contenant un conteneur Apache.
3. Déploiement du service Fargate.
4. Récupération de l’IP publique pour tester l’application.
5. Nettoyage complet des ressources.

---

## Remarque

Ce lab doit être réalisé dans **PowerShell** avec AWS CLI installé et configuré.

## Création du cluster ECS (Fargate)

Le cluster est l'infrastructure logique qui va accueillir les services.

```PowerShell
aws ecs create-cluster `
  --cluster-name fargate-cluster `
  --settings name=containerInsights,value=enabled `
  --tags key=Environnement,value=Dev key=Projet,value=Demo
```

![image](https://hackmd.io/_uploads/B1vaI6BFZl.png)
![image](https://hackmd.io/_uploads/B1ONP6BFWe.png)

## Créer le fichier JSON localement

C'est le "plan" qui décrit le conteneur (image Apache, CPU, RAM).

```PowerShell
@"
{
  "family": "sample-fargate",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "fargate-app",
      "image": "public.ecr.aws/docker/library/httpd:latest",
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "entryPoint": ["sh", "-c"],
      "command": [
        "/bin/sh -c \"echo '<html><head><title>Amazon ECS Sample App</title></head><body><h1>Amazon ECS Sample App</h1><h2>Congratulations!</h2><p>Your application is now running on a container in Amazon ECS.</p></body></html>' > /usr/local/apache2/htdocs/index.html && httpd-foreground\""
      ]
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512"
}
"@ | Out-File -FilePath taskdef.json -Encoding ascii
```

![image](https://hackmd.io/_uploads/HypLDprFZx.png)

## Enregistrer la définition de tâche

On envoie le plan au contrôleur ECS pour qu'il sache quoi déployer.

```PowerShell
aws ecs register-task-definition --cli-input-json file://taskdef.json

```

![image](https://hackmd.io/_uploads/HJOxu6rtbg.png)
![image](https://hackmd.io/_uploads/Bk9GtaBYWx.png)

## Création du service ECS (Fargate)

ECS a besoin de savoir où placer la tâche physiquement.

### Création du groupe de sécurité et récupération des sous réseaux

```PowerShell
# Récupérer les sous-réseaux
$subnets = (aws ec2 describe-subnets --query "Subnets[*].SubnetId" --output text).Replace("`t", ",")

# Créer le groupe de sécurité
$sg = aws ec2 create-security-group `
  --group-name fargate-sg `
  --description "SG for Fargate service" `
  --vpc-id (aws ec2 describe-vpcs --query "Vpcs[0].VpcId" --output text) `
  --query "GroupId" --output text

# Autoriser le trafic Web (Port 80)
aws ec2 authorize-security-group-ingress --group-id $sg --protocol tcp --port 80 --cidr 0.0.0.0/0
```

![image](https://hackmd.io/_uploads/Hk8vOaBKZl.png)
![image](https://hackmd.io/_uploads/r1Hsu6HFWe.png)

### Créer le service Fargate

Le service va lancer la tâche et s'assurer qu'elle reste active.

```PowerShell
aws ecs create-service `
  --cluster fargate-cluster `
  --service-name fargate-service `
  --task-definition sample-fargate `
  --desired-count 1 `
  --launch-type FARGATE `
  --network-configuration "awsvpcConfiguration={subnets=[$subnets],securityGroups=[$sg],assignPublicIp=ENABLED}"

```

![image](https://hackmd.io/_uploads/HkfA_aHYZl.png)
![image](https://hackmd.io/_uploads/HJhlF6BtWx.png)

### Récupérer l’IP publique de la tâche

Une fois le service lancé, récupère l'IP pour tester dans le navigateur.

```PowerShell
# 1. 'ID de l'interface réseau associée à la tâche
$eniId = aws ecs describe-tasks `
  --cluster fargate-cluster `
  --tasks (aws ecs list-tasks --cluster fargate-cluster --query "taskArns[0]" --output text) `
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" `
  --output text

# 2.  afficher l'IP publique directement
aws ec2 describe-network-interfaces `
  --network-interface-ids $eniId `
  --query "NetworkInterfaces[0].Association.PublicIp" `
  --output text

```

![image](https://hackmd.io/_uploads/BkjUqTBKZe.png)

### Se connecter à l'interface avec l'IP publique

![image](https://hackmd.io/_uploads/SygYqpSFZg.png)

## Nettoyage

```Powershell
# arrêter les tâches
aws ecs update-service --cluster fargate-cluster --service fargate-service --desired-count 0 ; `
# supprime le service
aws ecs delete-service --cluster fargate-cluster --service fargate-service --force ; `
# supprime le cluster
aws ecs delete-cluster --cluster fargate-cluster

```

![image](https://hackmd.io/_uploads/SJV09TSKZg.png)
![image](https://hackmd.io/_uploads/HkueipStbe.png)

## Supprimer le Groupe de sécurité

```PowerShell
aws ec2 delete-security-group --group-id $sg
```

![image](https://hackmd.io/_uploads/ryC9j6SKZe.png)
![image](https://hackmd.io/_uploads/Byk3jaBKZg.png)
