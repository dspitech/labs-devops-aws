# LAB : ClouFormation (Json - YAML)

## Lien : [aws-cloudformation-templates](https://github.com/aws-cloudformation/aws-cloudformation-templates/tree/main)

## Projet : Déployer et héberger Wordpress dans AWS

## Architecture monolithique

![Arcchitecture_monolithique (1)](https://hackmd.io/_uploads/r10pNl15-g.png)

### Fichier YAML Partiel : test déploiement Wordpress

![image](https://hackmd.io/_uploads/HyAuM-k9Ze.png)

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: 'WordPress Free Tier - Final Version Dynamique'

Parameters:
  DBPassword:
    Type: String
    NoEcho: true
    Default: "PassSafe2026"
    Description: Mot de passe de la base de données.
  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64'

Resources:
  WPS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "wp-free-lab-${AWS::AccountId}-storage"

  DBPasswordParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: "/wp/db/password"
      Type: "String"
      Value: !Ref DBPassword

  WPWebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Allow HTTP and SSH"
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0}
        - {IpProtocol: tcp, FromPort: 22, ToPort: 22, CidrIp: 0.0.0.0/0}

  WPDBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Allow MySQL from Web SG"
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !GetAtt WPWebSecurityGroup.GroupId

  WPIAMRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: {Service: [ec2.amazonaws.com]}
            Action: ["sts:AssumeRole"]
      Policies:
        - PolicyName: WordPressAccessPolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: ["s3:*", "ssm:GetParameter", "ssm:GetParameters"]
                Resource: "*"

  WPInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles: [!Ref WPIAMRole]

  WPDatabase:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: !Sub "rds-wp-free-${AWS::StackName}"
      DBName: "wordpressdb"
      AllocatedStorage: "20"
      DBInstanceClass: "db.t3.micro"
      Engine: "mysql"
      MasterUsername: "admin"
      MasterUserPassword: !Ref DBPassword
      VPCSecurityGroups: [!GetAtt WPDBSecurityGroup.GroupId]
      BackupRetentionPeriod: 0
      MultiAZ: false
      PubliclyAccessible: false

  WPInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "t3.micro"
      # UTILISATION DE L'AMI DYNAMIQUE ICI
      ImageId: !Ref LatestAmiId
      SecurityGroupIds:
        - !GetAtt WPWebSecurityGroup.GroupId
      IamInstanceProfile: !Ref WPInstanceProfile
      Tags:
        - {Key: Name, Value: "WP-Free-Lab"}
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          dnf update -y
          dnf install httpd php8.2 php8.2-mysqlnd mariadb105 -y
          cd /var/www/html
          wget https://wordpress.org/latest.tar.gz
          tar -xzf latest.tar.gz --strip-components=1
          cp wp-config-sample.php wp-config.php
          sed -i "s/database_name_here/wordpressdb/" wp-config.php
          sed -i "s/username_here/admin/" wp-config.php
          sed -i "s/password_here/${DBPassword}/" wp-config.php
          sed -i "s/localhost/${WPDatabase.Endpoint.Address}/" wp-config.php
          chown -R apache:apache /var/www/html
          systemctl enable --now httpd

Outputs:
  WebsiteURL:
    Description: "URL de votre site WordPress"
    Value: !Sub "http://${WPInstance.PublicIp}"
```

## Ce que fait ce script

Ce script déploie automatiquement (un exemple de configuration CloudFormation) un site **WordPress** dans AWS .  
Il crée :

- Une instance EC2 pour le site web
- Une base de données MySQL RDS
- Un bucket S3 pour le stockage
- Les groupes de sécurité et un rôle IAM nécessaires
- La configuration complète de WordPress via un script UserData

---

### Déploiement

- Étape 1 : Préparation du fichier YAML
- Copiez le code et enregistrez-le sous le `nom wordpress-lab.yaml`.
  ![image](https://hackmd.io/_uploads/HJFmulyqbx.png)

- Étape 2 : Configuration CloudFormation
  ![image](https://hackmd.io/_uploads/SJS2ue1qZe.png)
- Création d'une pile et uploader le fichier
  ![image](https://hackmd.io/_uploads/Bk2XKxkq-g.png)
  ![image](https://hackmd.io/_uploads/B1YitgkqZl.png)
  ![image](https://hackmd.io/_uploads/Sk6RYx1qbl.png)
  ![image](https://hackmd.io/_uploads/rJ1eqlJcbg.png)
  ![image](https://hackmd.io/_uploads/BJf-cgJ5Zl.png)
  ![image](https://hackmd.io/_uploads/B1PGqeyqZe.png)
  ![image](https://hackmd.io/_uploads/SkgVqxk9Zl.png)
  ![image](https://hackmd.io/_uploads/HkvH6lkqZl.png)

- Fin du déploiement (Voir les ressources) - Groupe de sécurité
  ![image](https://hackmd.io/_uploads/HkO9axy5bl.png)
  ![image](https://hackmd.io/_uploads/SyCnpgJcWx.png)
  ![image](https://hackmd.io/_uploads/rJkRplk9-e.png) - Base de données RDS
  ![image](https://hackmd.io/_uploads/B1HZ0lk9-g.png) - Bucket S3
  ![image](https://hackmd.io/_uploads/ByUrAxy9-x.png)

      - EC2

  ![image](https://hackmd.io/_uploads/BJFpf-yc-l.png)

### Accès au site Web

![image](https://hackmd.io/_uploads/Sk8EXZ1qWg.png)
![image](https://hackmd.io/_uploads/HkHv7WJ5Zg.png)
![image](https://hackmd.io/_uploads/H1OdXZJ9Zg.png)
![image](https://hackmd.io/_uploads/ry2Km-ycbx.png)
![image](https://hackmd.io/_uploads/SJ3cmWyqZg.png)
![image](https://hackmd.io/_uploads/ryTi7-yq-g.png)

### Fichier Json complet : architetcure monolithique

Fichier : `arcitecturemonolithique.yaml`

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Architecture WordPress monolithique avec VPC, EC2, RDS, S3, CloudFront et Route53

Parameters:
  DBPassword:
    Type: String
    NoEcho: true
    Description: Mot de passe RDS

  DomainName:
    Type: String
    Default: exemple.com

  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64'

Resources:

# ---------------- VPC ----------------
  WPVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 172.16.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref WPVPC
      InternetGatewayId: !Ref InternetGateway

# ---------------- SUBNETS ----------------
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref WPVPC
      AvailabilityZone: !Select [0, !GetAZs ""]
      CidrBlock: 172.16.1.0/24
      MapPublicIpOnLaunch: true

  PrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref WPVPC
      AvailabilityZone: !Select [1, !GetAZs ""]
      CidrBlock: 172.16.2.0/24

# ---------------- ROUTE TABLE ----------------
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref WPVPC

  DefaultRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  RouteAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

# ---------------- SECURITY GROUPS ----------------
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web access
      VpcId: !Ref WPVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: MySQL access
      VpcId: !Ref WPVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref WebSecurityGroup

# ---------------- IAM ----------------
  WPRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonS3FullAccess
        - arn:aws:iam::aws:policy/AmazonRekognitionFullAccess

  WPInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref WPRole

# ---------------- S3 ----------------
  WPBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "wp-storage-${AWS::AccountId}-${AWS::Region}"

# ---------------- SECRETS MANAGER ----------------
  WPSecrets:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: !Sub "WPSecrets-${AWS::StackName}"
      SecretString: !Sub '{"username":"admin","password":"${DBPassword}"}'

# ---------------- RDS ----------------
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: RDS subnets
      SubnetIds:
        - !Ref PublicSubnet
        - !Ref PrivateSubnet

  WPDatabase:
    Type: AWS::RDS::DBInstance
    Properties:
      DBName: wordpressdb
      Engine: mysql
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !GetAtt DBSecurityGroup.GroupId
      PubliclyAccessible: false

# ---------------- EC2 WORDPRESS ----------------
  WPInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref LatestAmiId
      IamInstanceProfile: !Ref WPInstanceProfile
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref WebSecurityGroup

      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          dnf update -y
          dnf install httpd php8.2 php8.2-mysqlnd mariadb105 -y

          systemctl enable httpd
          systemctl start httpd

          cd /var/www/html

          wget https://wordpress.org/latest.tar.gz
          tar -xzf latest.tar.gz --strip-components=1

          cp wp-config-sample.php wp-config.php

          sed -i "s/database_name_here/wordpressdb/" wp-config.php
          sed -i "s/username_here/admin/" wp-config.php
          sed -i "s/password_here/${DBPassword}/" wp-config.php
          sed -i "s/localhost/${WPDatabase.Endpoint.Address}/" wp-config.php

          chown -R apache:apache /var/www/html

# ---------------- CLOUDFRONT ----------------
  CloudFrontDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Enabled: true
        Origins:
          - DomainName: !GetAtt WPInstance.PublicDnsName
            Id: EC2Origin
            CustomOriginConfig:
              OriginProtocolPolicy: http-only

        DefaultCacheBehavior:
          TargetOriginId: EC2Origin
          ViewerProtocolPolicy: allow-all
          ForwardedValues:
            QueryString: true
            Cookies:
              Forward: all

# ---------------- ROUTE53 ----------------
  WPRecordSet:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneName: !Sub "${DomainName}."
      Name: !Sub "www.${DomainName}"
      Type: A
      AliasTarget:
        DNSName: !GetAtt CloudFrontDistribution.DomainName
        HostedZoneId: Z2FDTNDATAQYW2

# ---------------- OUTPUTS ----------------
Outputs:

  WebsiteURL:
    Value: !Sub "http://www.${DomainName}"

  CloudFrontDomain:
    Value: !GetAtt CloudFrontDistribution.DomainName

  BucketName:
    Value: !Ref WPBucket
```

## Architecture MicroService

![Arcchitecture_microservice](https://hackmd.io/_uploads/r1pzBZyc-g.png)

Fichier : `arcitecturemicroservice.yaml`

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Architecture WordPress Microservices (ECS, RDS Multi-AZ, Redis, EFS, CloudFront, Route53)

Parameters:
  DBPassword:
    Type: String
    NoEcho: true
  DomainName:
    Type: String

Resources:

# ---------------- VPC ----------------
  WPVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref WPVPC
      InternetGatewayId: !Ref InternetGateway

# ---------------- SUBNETS ----------------
  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref WPVPC
      AvailabilityZone: !Select [0, !GetAZs ""]
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true

  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref WPVPC
      AvailabilityZone: !Select [1, !GetAZs ""]
      CidrBlock: 10.0.2.0/24
      MapPublicIpOnLaunch: true

  PrivateSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref WPVPC
      AvailabilityZone: !Select [0, !GetAZs ""]
      CidrBlock: 10.0.3.0/24

  PrivateSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref WPVPC
      AvailabilityZone: !Select [1, !GetAZs ""]
      CidrBlock: 10.0.4.0/24

# ---------------- SECURITY GROUPS ----------------
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web access
      VpcId: !Ref WPVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  DBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: MySQL access
      VpcId: !Ref WPVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref WebSG

  StorageSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Storage access
      VpcId: !Ref WPVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 2049
          ToPort: 2049
          CidrIp: 10.0.0.0/16

# ---------------- EFS ----------------
  WordPressEFS:
    Type: AWS::EFS::FileSystem

  EFSMountTargetA:
    Type: AWS::EFS::MountTarget
    Properties:
      FileSystemId: !Ref WordPressEFS
      SubnetId: !Ref PrivateSubnetA
      SecurityGroups:
        - !Ref StorageSG

  EFSMountTargetB:
    Type: AWS::EFS::MountTarget
    Properties:
      FileSystemId: !Ref WordPressEFS
      SubnetId: !Ref PrivateSubnetB
      SecurityGroups:
        - !Ref StorageSG

# ---------------- REDIS ----------------
  RedisSubnetGroup:
    Type: AWS::ElastiCache::CacheSubnetGroup
    Properties:
      Description: Redis subnets
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB

  RedisCluster:
    Type: AWS::ElastiCache::ReplicationGroup
    Properties:
      ReplicationGroupDescription: Redis cache
      Engine: redis
      CacheNodeType: cache.t3.micro
      NumCacheClusters: 2
      AutomaticFailoverEnabled: true
      CacheSubnetGroupName: !Ref RedisSubnetGroup

# ---------------- RDS ----------------
  RDSSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: RDS subnets
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB

  RDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: mysql
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      MultiAZ: true
      VPCSecurityGroups:
        - !GetAtt DBSG.GroupId
      DBSubnetGroupName: !Ref RDSSubnetGroup

# ---------------- LOAD BALANCER ----------------
  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets:
        - !Ref PublicSubnetA
        - !Ref PublicSubnetB
      SecurityGroups:
        - !Ref WebSG

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Port: 80
      Protocol: HTTP
      VpcId: !Ref WPVPC
      TargetType: ip

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup

# ---------------- ECS ----------------
  ECSCluster:
    Type: AWS::ECS::Cluster

# ---------------- CLOUDFRONT ----------------
  CloudFront:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Enabled: true
        Origins:
          - DomainName: !GetAtt ALB.DNSName
            Id: ALBOrigin
            CustomOriginConfig:
              OriginProtocolPolicy: http-only
        DefaultCacheBehavior:
          TargetOriginId: ALBOrigin
          ViewerProtocolPolicy: redirect-to-https
          ForwardedValues:
            QueryString: true
            Cookies:
              Forward: all

# ---------------- ROUTE53 ----------------
  WPRecordSet:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneName: !Sub "${DomainName}."
      Name: !Sub "www.${DomainName}"
      Type: A
      AliasTarget:
        DNSName: !GetAtt CloudFront.DomainName
        HostedZoneId: Z2FDTNDATAQYW2

Outputs:
  WebsiteURL:
    Value: !Sub "https://www.${DomainName}"

  CloudFrontURL:
    Value: !GetAtt CloudFront.DomainName
```

## Architetcure Événementielle

![Architecture_Evenmentielle](https://hackmd.io/_uploads/BJGt8by5Zg.png)

Fichier : `arcitectureevenmentielle.yaml`

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: WordPress scalable architecture (VPC, ALB, Auto Scaling, RDS)

Parameters:
  DBPassword:
    Type: String
    NoEcho: true

Resources:

# ---------------- VPC ----------------
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

# ---------------- SUBNETS ----------------
  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs ""]
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true

  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs ""]
      CidrBlock: 10.0.2.0/24
      MapPublicIpOnLaunch: true

  PrivateSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs ""]
      CidrBlock: 10.0.3.0/24

  PrivateSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs ""]
      CidrBlock: 10.0.4.0/24

# ---------------- ROUTES ----------------
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnetRouteA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetA
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetRouteB:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetB
      RouteTableId: !Ref PublicRouteTable

# ---------------- SECURITY GROUPS ----------------
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web access
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  DBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: DB access
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref WebSG

# ---------------- LOAD BALANCER ----------------
  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets:
        - !Ref PublicSubnetA
        - !Ref PublicSubnetB
      SecurityGroups:
        - !Ref WebSG

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VPC
      TargetType: instance
      HealthCheckPath: /

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup

# ---------------- IAM ROLE ----------------
  WPRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole

  WPInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref WPRole

# ---------------- LAUNCH TEMPLATE ----------------
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: ami-0c55b159cbfafe1f0
        InstanceType: t3.micro
        SecurityGroupIds:
          - !Ref WebSG
        IamInstanceProfile:
          Name: !Ref WPInstanceProfile
        UserData:
          Fn::Base64: |
            #!/bin/bash
            yum update -y
            amazon-linux-extras install php7.4 -y
            yum install -y httpd mysql
            systemctl enable httpd
            systemctl start httpd

# ---------------- AUTO SCALING ----------------
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB
      MinSize: 2
      MaxSize: 4
      TargetGroupARNs:
        - !Ref TargetGroup
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber

# ---------------- RDS ----------------
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: DB subnet
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB

  WordPressDB:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: mysql
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      VPCSecurityGroups:
        - !GetAtt DBSG.GroupId
      DBSubnetGroupName: !Ref DBSubnetGroup
      PubliclyAccessible: false

# ---------------- OUTPUT ----------------
Outputs:
  LoadBalancerURL:
    Description: URL du site WordPress
    Value: !Sub "http://${ALB.DNSName}"
```
