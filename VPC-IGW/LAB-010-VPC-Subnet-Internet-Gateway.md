# LAB : AWS VPC, Subnet et Internet Gateway

Ce lab montre comment créer un VPC privé, un subnet public, et configurer un Internet Gateway avec des routes vers Internet.

## Créer un nouveau VPC

```Bash
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --query 'Vpc.VpcId' --output text)

echo "VPC créé : $VPC_ID"
```

![image](https://hackmd.io/_uploads/rJHqDre5Wl.png)

![image](https://hackmd.io/_uploads/Bya2wrecZx.png)

### Explications

- **10.0.0.0/16** : plage d’adresses IP privées pour le VPC
- On récupère automatiquement le **VPC ID** pour les étapes suivantes

## Créer un Subnet public

```Bash
SUBNET_ID=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.1.0/24 \
    --availability-zone eu-west-3a \
    --query 'Subnet.SubnetId' --output text)

echo "Subnet public créé : $SUBNET_ID"
```

![image](https://hackmd.io/_uploads/SJLJ_Beqbl.png)

![image](https://hackmd.io/_uploads/S13m_rl9Ze.png)

### Explications

- **10.0.1.0/24** : plage IP du subnet
- **availability-zone eu-west-3a** : zone AWS
- **SUBNET_ID** est utilisé pour la table de routage

## Créer un Internet Gateway (IGW)

```Bash
IGW_ID=$(aws ec2 create-internet-gateway \
    --query 'InternetGateway.InternetGatewayId' --output text)

echo "Internet Gateway créé : $IGW_ID"
```

![image](https://hackmd.io/_uploads/Bk5IOrx9Zl.png)

![image](https://hackmd.io/_uploads/Bkk9uBe5-l.png)

## Attacher l’IGW au VPC

```Bash
aws ec2 attach-internet-gateway \
    --vpc-id $VPC_ID \
    --internet-gateway-id $IGW_ID

echo "IGW attaché au VPC"
```

![image](https://hackmd.io/_uploads/BJYi_Hl9-l.png)

![image](https://hackmd.io/_uploads/B17Rurgc-l.png)

## Créer une table de routage

```Bash
ROUTE_TABLE_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --query 'RouteTable.RouteTableId' --output text)

echo "Table de routage créée : $ROUTE_TABLE_ID"
```

![image](https://hackmd.io/_uploads/H1_btBl9We.png)

![image](https://hackmd.io/_uploads/SJeDKHxqWl.png)

## Ajouter une route vers l’IGW

```Bash
aws ec2 create-route \
    --route-table-id $ROUTE_TABLE_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id $IGW_ID

echo "Route 0.0.0.0/0 vers IGW ajoutée"
```

![image](https://hackmd.io/_uploads/B1iitrlqbe.png)

## Associer le subnet à la table de routage

```Bash
aws ec2 associate-route-table \
    --subnet-id $SUBNET_ID \
    --route-table-id $ROUTE_TABLE_ID

echo "Subnet associé à la table de routage"
```

![image](https://hackmd.io/_uploads/HylbqBe5Wl.png)

![image](https://hackmd.io/_uploads/BksmqHx9Zg.png)

## Lancer une instance EC2 t2.micro

### Création de la clé SSH

```Bash
# Nom de la clé SSH
KEY_NAME="DevOpsKeyTP"

# Créer la clé SSH
aws ec2 create-key-pair \
    --key-name $KEY_NAME \
    --query 'KeyMaterial' \
    --output text > ${KEY_NAME}.pem

chmod 400 ${KEY_NAME}.pem
echo "Clé SSH créée : ${KEY_NAME}.pem"

# Vérifier que le fichier existe
ls -l ${KEY_NAME}.pem
```

![image](https://hackmd.io/_uploads/rkqo0Se5Ze.png)

![image](https://hackmd.io/_uploads/SyY6RHxcWg.png)

## Création du groupe de sécurité

```Bash
SG_NAME="DevOpsSG"
SG_DESC="Security group pour TP EC2 SSH"

# Création du Security Group
SG_ID=$(aws ec2 create-security-group \
    --group-name $SG_NAME \
    --description "$SG_DESC" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

echo "Security Group créé : $SG_ID"
```

![image](https://hackmd.io/_uploads/SyHdJLgqZl.png)

![image](https://hackmd.io/_uploads/SyjiyUeq-l.png)

### Autoriser le port 22 SSH

```Bash
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0

echo "Port SSH 22 ouvert pour le SG $SG_NAME"
```

![image](https://hackmd.io/_uploads/rJqkxUgq-l.png)

![image](https://hackmd.io/_uploads/Sk_WeUx5Zl.png)

### Création de l'instance

```Bash
# Variables
VPC_ID="vpc-08b2966331dcea02a"
SUBNET_ID="subnet-0d6496aa540a98f07"
SG_ID="sg-084f126558c0e59c4"
KEY_NAME="DevOpsKeyTP"
AMI_ID="ami-05d43d5e94bb6eb95"
INSTANCE_NAME="InstanceTP"

# Lancer l'instance EC2 t2.micro
INSTANCE_ID=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --count 1 \
    --instance-type t2.micro \
    --key-name $KEY_NAME \
    --security-group-ids $SG_ID \
    --subnet-id $SUBNET_ID \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$INSTANCE_NAME}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

echo "Instance EC2 t2.micro créée : $INSTANCE_ID"

# Attendre que l’instance soit running
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance en cours d’exécution"

# Afficher les détails
aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name,Subnet:SubnetId,SG:SecurityGroups[*].GroupName,PublicIP:PublicIpAddress}" \
    --output table
```

![image](https://hackmd.io/_uploads/BJnOGUecZe.png)

![image](https://hackmd.io/_uploads/BJ-9MUx9Ze.png)

![image](https://hackmd.io/_uploads/HklhG8gcbg.png)

## Vérification finale

```Bash
# --- Vérification finale des ressources ---

echo "Ressources créées :"
echo "VPC             : $VPC_ID"
echo "Subnet          : $SUBNET_ID"
echo "Internet Gateway: $IGW_ID"
echo "Route Table     : $ROUTE_TABLE_ID"
echo "Security Group  : $SG_ID"
echo "Instance EC2    : $INSTANCE_ID"

# Détails de l'instance EC2
aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query "Reservations[].Instances[].{
        ID: InstanceId,
        State: State.Name,
        Subnet: SubnetId,
        SecurityGroups: SecurityGroups[*].GroupName,
        PublicIP: PublicIpAddress
    }" \
    --output table
```

![image](https://hackmd.io/_uploads/HyBzmLe5Ze.png)

![image](https://hackmd.io/_uploads/HyEQ7Il5bg.png)

## Étapes pour créer et associer une Elastic IP

### Créer une Elastic IP

```Bash
ALLOC_ID=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
echo "Elastic IP créée avec AllocationId : $ALLOC_ID"
```

![image](https://hackmd.io/_uploads/r1ukV8lcWg.png)

### Associer l’Elastic IP à ton instance EC2

```Bash
aws ec2 associate-address --instance-id $INSTANCE_ID --allocation-id $ALLOC_ID
echo "Elastic IP associée à l'instance $INSTANCE_ID"
```

![image](https://hackmd.io/_uploads/rJsdNLg5-e.png)

### Récupérer l’IP publique

```Bash
PUBLIC_IP=$(aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query "Reservations[].Instances[].PublicIpAddress" --output text)
echo "IP publique de l'instance : $PUBLIC_IP"
```

![image](https://hackmd.io/_uploads/Bk124Ixcbe.png)

## Connexon SSH

```Bash
ssh -i "DevOpsKeyTP.pem" ec2-user@13.36.5.162
```

![image](https://hackmd.io/_uploads/BkMzSIe9bx.png)

## Nettoyage

Avant de supprimer, il faut récupèrer les informations des ressources.

```Bash
VPC_ID=$(aws ec2 describe-vpcs --query "Vpcs[0].VpcId" --output text) && \
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text) && \
SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=DevOpsSG" --query "SecurityGroups[0].GroupId" --output text) && \
IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query "InternetGateways[0].InternetGatewayId" --output text) && \
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=subnet-id,Values=$SUBNET_ID" "Name=instance-state-name,Values=running,pending,stopped" --query "Reservations[0].Instances[0].InstanceId" --output text) && \
KEY_NAME="DevOpsKeyTP" && \
ALLOC_ID=$(aws ec2 describe-addresses --filters "Name=domain,Values=vpc" --query "Addresses[0].AllocationId" --output text) && \
echo -e "\n--- CONFIGURATION DÉTECTÉE ---\nVPC: $VPC_ID\nSUBNET: $SUBNET_ID\nSG: $SG_ID\nIGW: $IGW_ID\nINSTANCE: $INSTANCE_ID\nKEY: $KEY_NAME\nALLOC_ID: $ALLOC_ID\n------------------------------"

```

![image](https://hackmd.io/_uploads/HktH8Le5-g.png)

Pour bien néttoyer les ressources, remplace les bons Id à leur place.

```Bash
VPC_ID="vpc-08b2966331dcea02a"
SUBNET_ID="subnet-0d6496aa540a98f07"
SG_ID="sg-084f126558c0e59c4"
IGW_ID="igw-00194e1eac7015200"
INSTANCE_ID="i-0837970334062c02a"
KEY_NAME="DevOpsKeyTP"
ALLOC_ID="eipalloc-0e6bd0bc63a31dc9"

# On récupère dynamiquement la table de routage et le volume EBS
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query "RouteTables[0].RouteTableId" --output text)
VOL_ID=$(aws ec2 describe-volumes --filters "Name=attachment.instance-id,Values=$INSTANCE_ID" --query "Volumes[0].VolumeId" --output text)

echo "--- Variables chargées avec succès ---"
```

### Étape : Terminer l'Instance

```Bash
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

**Important** : On doit attendre qu'elle soit totalement supprimée (terminated) avant de passer à la suite, sinon le réseau restera bloqué.

### Étape : Libérer l'Elastic IP

```Bash
aws ec2 release-address --allocation-id $ALLOC_ID
echo "Elastic IP libérée."
```

### Étape : Supprimer l'Internet Gateway (IGW)

On ne peut pas supprimer une passerelle si elle est encore branchée au VPC. On détache, puis on supprime.

```Bash
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
echo "Internet Gateway supprimée."
```

### Étape : Supprimer le Security Group et le Subnet

```Bash
# Supprimer le securty-group
aws ec2 delete-security-group --group-id $SG_ID
echo "Security Group supprimé."

# Supprimer le sous-réseau
aws ec2 delete-subnet --subnet-id $SUBNET_ID
echo "Subnet supprimé."
```

### Étape : Supprimer le VPC

```Bash
aws ec2 delete-vpc --vpc-id $VPC_ID
echo "VPC supprimé."
```

### Étape : Nettoyage de la clé

```Bash
aws ec2 delete-key-pair --key-name $KEY_NAME
rm -f ${KEY_NAME}.pem
echo "Clé SSH supprimée."
```

## Vérification finale

```Bash
echo "--- VÉRIFICATION FINALE ---"

echo -e "\n1. Instances (Statut doit être 'terminated' ou vide) :"
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running,pending,stopped,stopping" --query "Reservations[*].Instances[*].{ID:InstanceId,State:State.Name}" --output table

echo -e "\n2. Elastic IPs (Devrait être vide) :"
aws ec2 describe-addresses --query "Addresses[*].{IP:PublicIp,AllocationId:AllocationId}" --output table

echo -e "\n3. Volumes EBS (Devrait être vide) :"
aws ec2 describe-volumes --query "Volumes[*].{ID:VolumeId,State:State}" --output table

echo -e "\n4. Snapshots & AMIs (Tes sauvegardes) :"
aws ec2 describe-snapshots --owner-ids self --query "Snapshots[*].SnapshotId" --output table
aws ec2 describe-images --owners self --query "Images[*].ImageId" --output table

echo -e "\n5. Réseau (VPC, Subnet, SG - Devrait être vide) :"
aws ec2 describe-vpcs --filters "Name=is-default,Values=false" --query "Vpcs[*].VpcId" --output table
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[*].SubnetId" --output table
aws ec2 describe-security-groups --filters "Name=group-name,Values=DevOpsSG" --query "SecurityGroups[*].GroupId" --output table

echo -e "\n6. Clé SSH (AWS) :"
aws ec2 describe-key-pairs --filters "Name=key-name,Values=DevOpsKeyTP" --query "KeyPairs[*].KeyName" --output table

echo -e "\n--- FIN DU SCAN ---"
```
