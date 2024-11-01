# Projeto Final AWS re/Start Escola da Nuvem BRSAO142 

<div align="center">
  <img src="imagens/EDN-capa.png"/>
</div>

---

## Objetivo

<div align="center">
  <img src="imagens/EDN-objetivo.pptx.png"/>
</div>

---

## Cenário Atual

<div align="center">
  <img src="imagens/EDN-cenario-atual.PNG"/>
</div>

---

## Solução Proposta

<div align="center">
  <img src="imagens/EDN-solucao-final.png"/>
</div>

#

Youtube

<p align="center">
  <a href="https://www.youtube.com/watch?v=vgSRoiUvIHM">
    <img src="https://img.youtube.com/vi/vgSRoiUvIHM/0.jpg" alt="YouTube Video" width="400"/>
  </a>
</p>

---
## Template CloudFormation
Exemplo de template para implementação da arquitetura com CloudFormation:

```yml
AWSTemplateFormatVersion: '2010-09-09'
Description: "Template do CloudFormation para criar arquitetura para o projeto final da Escola da Nuvem."

Parameters:
  DBInstanceClass:
    Type: String
    Default: db.t3.micro
    Description: "A classe de instância do DB para o banco de dados RDS."

  InstanceType:
    Type: String
    Default: t2.micro
    Description: "O tipo de instância para o servidor web."

  AMIId:
    Type: String
    Default: ami-0ebfd941bbafe70c6
    Description: "Digite um ID de AMI que esteja disponível na sua região."

  MasterUsername:
    Type: String
    Default: admin
    Description: "O nome de usuário master para o banco de dados RDS."

  MasterUserPassword:
    Type: String
    NoEcho: true
    Description: "Insira a senha do usuário master para o banco de dados RDS."

  BucketUniqueSuffix:
    Type: String
    Default: "unique-id"
    Description: "Um sufixo único para o nome do bucket S3."

Resources:
  # VPC
  VPC-EDN:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: VPC-EDN

  # Sub-redes
  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC-EDN
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: PublicSubnetA

  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC-EDN
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [ 1, !GetAZs '' ]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: PublicSubnetB

  PrivateSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC-EDN
      CidrBlock: 10.0.3.0/24
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      Tags:
        - Key: Name
          Value: PrivateSubnetA

  PrivateSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC-EDN
      CidrBlock: 10.0.4.0/24
      AvailabilityZone: !Select [ 1, !GetAZs '' ]
      Tags:
        - Key: Name
          Value: PrivateSubnetB

  # Internet Gateway e Tabela de Roteamento para Sub-redes Públicas
  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC-EDN
      InternetGatewayId: !Ref InternetGateway

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC-EDN

  PublicRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnetARouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetA
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetBRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetB
      RouteTableId: !Ref PublicRouteTable

  # Grupos de Segurança
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Permitir acesso HTTP"
      VpcId: !Ref VPC-EDN
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  DatabaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Permitir acesso MySQL dos servidores web"
      VpcId: !Ref VPC-EDN
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref WebServerSecurityGroup

  # Load Balancer
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: MyALB
      Subnets:
        - !Ref PublicSubnetA
        - !Ref PublicSubnetB
      SecurityGroups:
        - !Ref WebServerSecurityGroup
      Scheme: internet-facing

  # Grupo de Auto Scaling e Configuração de Lançamento
  LaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !Ref AMIId 
      SecurityGroups:
        - !Ref WebServerSecurityGroup

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier:
        - !Ref PublicSubnetA
        - !Ref PublicSubnetB
      LaunchConfigurationName: !Ref LaunchConfiguration
      MinSize: 2
      MaxSize: 4
      DesiredCapacity: 2
      TargetGroupARNs:
        - !Ref TargetGroup

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: MyTargetGroup
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VPC-EDN
      HealthCheckProtocol: HTTP
      HealthCheckPort: 80

  # Banco de Dados RDS (Aurora)
  RDSInstance:
    Type: AWS::RDS::DBCluster
    Properties:
      Engine: aurora
      DBClusterIdentifier: MyAuroraCluster
      MasterUsername: !Ref MasterUsername
      MasterUserPassword: !Ref MasterUserPassword
      VpcSecurityGroupIds:
        - !Ref DatabaseSecurityGroup
      DBSubnetGroupName: !Ref DBSubnetGroup
      EnableIAMDatabaseAuthentication: true
      DatabaseName: MyDatabase
      StorageEncrypted: true
      BackupRetentionPeriod: 1
      MultiAZ: true

  RDSInstance1:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: !Ref DBInstanceClass
      Engine: aurora
      DBClusterIdentifier: !Ref RDSInstance
      PromotionTier: 1

  RDSInstance2:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: !Ref DBInstanceClass
      Engine: aurora
      DBClusterIdentifier: !Ref RDSInstance
      PromotionTier: 2

  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: "Sub-redes para RDS"
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB

  # Cluster ElastiCache com 2 nós
  ElastiCacheCluster:
    Type: AWS::ElastiCache::CacheCluster
    Properties:
      Engine: redis
      CacheNodeType: cache.t2.micro
      NumCacheNodes: 2 
      VpcSecurityGroupIds:
        - !Ref DatabaseSecurityGroup
      CacheSubnetGroupName: !Ref ElastiCacheSubnetGroup

  ElastiCacheSubnetGroup:
    Type: AWS::ElastiCache::SubnetGroup
    Properties:
      Description: "Grupo de sub-redes para ElastiCache"
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB

  # Registro DNS do Route 53
  DNSRecord:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneId: "<HOSTED_ZONE_ID>" 
      Name: www.example.com
      Type: A
      AliasTarget:
        DNSName: !GetAtt ApplicationLoadBalancer.DNSName
        HostedZoneId: !GetAtt ApplicationLoadBalancer.CanonicalHostedZoneId

  # Bucket S3
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "my-app-bucket-${BucketUniqueSuffix}-${AWS::StackName}-${AWS::Region}-${AWS::AccountId}"

  # Alarmes do CloudWatch para Instâncias EC2
  CPUUtilizationAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: "Alarme se o uso da CPU exceder 80%"
      Namespace: AWS/EC2
      MetricName: CPUUtilization
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AutoScalingGroup
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanOrEqualToThreshold
      AlarmActions: [] 

  # Função IAM para EC2 acessar S3 e CloudWatch
  EC2Role:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: S3CloudWatchAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:*
                  - cloudwatch:*
                Resource: "*"

Outputs:
  VPCId:
    Description: "ID da VPC"
    Value: !Ref VPC-EDN

  PublicSubnetAId:
    Description: "ID da Sub-rede Pública A"
    Value: !Ref PublicSubnetA

  PublicSubnetBId:
    Description: "ID da Sub-rede Pública B"
    Value: !Ref PublicSubnetB

  PrivateSubnetAId:
    Description: "ID da Sub-rede Privada A"
    Value: !Ref PrivateSubnetA

  PrivateSubnetBId:
    Description: "ID da Sub-rede Privada B"
    Value: !Ref PrivateSubnetB

  RDSInstanceEndpoint:
    Description: "Endpoint do RDS"
    Value: !GetAtt RDSInstance.Endpoint.Address

  ElastiCacheEndpoint:
    Description: "Endpoint do ElastiCache"
    Value: !GetAtt ElastiCacheCluster.RedisEndpoint.Address

  S3BucketName:
    Description: "Nome do Bucket S3"
    Value: !Ref S3Bucket
```

#

Ao criar a pilha do CloudFormation, passar a senha do usuário master para o banco de dados RDS através da AWS CLI:

```bash
aws cloudformation create-stack \
  --stack-name <nome-da-pilha> \
  --template-body file://<caminho-para-o-template>.yaml \
  --parameters ParameterKey=MasterUserPassword,ParameterValue=<sua-senha> \
  --capabilities CAPABILITY_IAM
```


