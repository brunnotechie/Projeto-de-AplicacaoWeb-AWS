# Documentação de Implementação - The Cloud Bootcamp Conference

## Índice
1. [Pré-requisitos](#pré-requisitos)
2. [Configuração Inicial](#configuração-inicial)
3. [Implementação da Infraestrutura](#implementação-da-infraestrutura)
4. [Configuração dos Serviços](#configuração-dos-serviços)
5. [Deploy da Aplicação](#deploy-da-aplicação)
6. [Monitoramento e Manutenção](#monitoramento-e-manutenção)
7. [Procedimentos de Backup](#procedimentos-de-backup)

## Pré-requisitos

### 1. Conta AWS e Permissões
```bash
# Instalar AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configurar credenciais
aws configure
```

### 2. Ferramentas Necessárias
```bash
# Instalar EB CLI
pip install awsebcli

# Instalar AWS SAM
pip install aws-sam-cli

# Instalar Terraform (opcional, para IaC)
wget https://releases.hashicorp.com/terraform/1.0.0/terraform_1.0.0_linux_amd64.zip
unzip terraform_1.0.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

## Configuração Inicial

### 1. Criar VPC e Subnets
```yaml
# vpc-template.yaml
Resources:
  ProjectVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: ConferenceVPC

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref ProjectVPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref ProjectVPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs ""]
      MapPublicIpOnLaunch: true
```

### 2. Configurar Security Groups
```yaml
# security-groups.yaml
Resources:
  ApplicationSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for application servers
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
```

## Implementação da Infraestrutura

### 1. Criar Tabela DynamoDB
```bash
# create-dynamodb.sh
aws dynamodb create-table \
  --table-name ConferenceParticipants \
  --attribute-definitions \
    AttributeName=email,AttributeType=S \
    AttributeName=registrationTimestamp,AttributeType=N \
  --key-schema \
    AttributeName=email,KeyType=HASH \
    AttributeName=registrationTimestamp,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

### 2. Configurar CloudFront
```yaml
# cloudfront-template.yaml
Resources:
  ConferenceDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Enabled: true
        DefaultCacheBehavior:
          TargetOriginId: ElasticBeanstalkEnvironment
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
          OriginRequestPolicyId: 88a5eaf4-2fd4-4709-b370-b4c650ea3fcf
        Origins:
          - DomainName: !Sub "${ElasticBeanstalkEnvironment}.elasticbeanstalk.com"
            Id: ElasticBeanstalkEnvironment
            CustomOriginConfig:
              OriginProtocolPolicy: https-only
```

### 3. Configurar Route 53
```bash
# configure-route53.sh
aws route53 create-hosted-zone \
  --name conference.example.com \
  --caller-reference $(date +%s)

aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "conference.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "CLOUDFRONT_ZONE_ID",
          "DNSName": "CLOUDFRONT_DOMAIN",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

## Configuração dos Serviços

### 1. Configurar Elastic Beanstalk
```yaml
# .elasticbeanstalk/config.yml
branch-defaults:
  main:
    environment: conference-prod
deploy:
  artifact: target/conference-app.jar
global:
  application_name: ConferenceApp
  default_region: us-east-1
  workspace_type: Application
option_settings:
  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 10
  aws:autoscaling:trigger:
    UpperThreshold: 70
    LowerThreshold: 30
    MeasureName: CPUUtilization
    Unit: Percent
    UpperBreachScaleIncrement: 2
    LowerBreachScaleIncrement: -1
```

### 2. Configurar CloudWatch
```bash
# configure-cloudwatch.sh
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPUUtilization \
  --alarm-description "CPU utilization exceeds 70%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:region:account-id:NotifyDevOps"
```

## Deploy da Aplicação

### 1. Preparar Aplicação
```bash
# build-and-deploy.sh
#!/bin/bash

# Compilar aplicação
mvn clean package

# Inicializar Elastic Beanstalk
eb init ConferenceApp --platform "Java 11" --region us-east-1

# Criar ambiente
eb create conference-prod \
  --instance_type t3.medium \
  --vpc.id vpc-xxxxx \
  --vpc.ec2subnets subnet-xxxxx,subnet-yyyyy \
  --vpc.elbsubnets subnet-xxxxx,subnet-yyyyy \
  --vpc.securitygroups sg-xxxxx \
  --scale 2

# Deploy da aplicação
eb deploy conference-prod
```

### 2. Configurar SSL/TLS
```bash
# configure-ssl.sh
aws acm request-certificate \
  --domain-name conference.example.com \
  --validation-method DNS \
  --region us-east-1
```

## Monitoramento e Manutenção

### 1. Configurar Dashboards
```bash
# create-dashboard.sh
aws cloudwatch put-dashboard \
  --dashboard-name ConferenceMetrics \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/EC2", "CPUUtilization"],
            ["AWS/ApplicationELB", "RequestCount"],
            ["AWS/DynamoDB", "ConsumedReadCapacityUnits"],
            ["AWS/DynamoDB", "ConsumedWriteCapacityUnits"]
          ],
          "period": 300,
          "stat": "Average",
          "region": "us-east-1",
          "title": "Conference Metrics"
        }
      }
    ]
  }'
```

### 2. Configurar Alertas
```yaml
# alerts-template.yaml
Resources:
  HighLatencyAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Response time is too high
      MetricName: Latency
      Namespace: AWS/ELB
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 1
      AlarmActions:
        - !Ref AlertingSNSTopic
      ComparisonOperator: GreaterThanThreshold
```

## Procedimentos de Backup

### 1. Configurar Backup do DynamoDB
```bash
# configure-backup.sh
aws dynamodb update-continuous-backups \
  --table-name ConferenceParticipants \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```

### 2. Configurar Retenção de Logs
```yaml
# logs-retention.yaml
Resources:
  LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /aws/elasticbeanstalk/ConferenceApp
      RetentionInDays: 30
```

## Scripts de Manutenção

### 1. Script de Limpeza
```python
# cleanup.py
import boto3

def cleanup_old_data():
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('ConferenceParticipants')
    
    # Limpar registros antigos
    response = table.scan(
        FilterExpression='registrationTimestamp < :old_date',
        ExpressionAttributeValues={
            ':old_date': int(time.time()) - (30 * 24 * 60 * 60)  # 30 dias
        }
    )
    
    with table.batch_writer() as batch:
        for item in response['Items']:
            batch.delete_item(
                Key={
                    'email': item['email'],
                    'registrationTimestamp': item['registrationTimestamp']
                }
            )
```

### 2. Script de Monitoramento
```python
# monitor.py
import boto3
import datetime

def check_system_health():
    cloudwatch = boto3.client('cloudwatch')
    
    # Verificar métricas do sistema
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/EC2',
        MetricName='CPUUtilization',
        Dimensions=[{'Name': 'AutoScalingGroupName', 'Value': 'ConferenceApp-ASG'}],
        StartTime=datetime.datetime.utcnow() - datetime.timedelta(hours=1),
        EndTime=datetime.datetime.utcnow(),
        Period=300,
        Statistics=['Average']
    )
    
    # Análise e alerta
    for datapoint in response['Datapoints']:
        if datapoint['Average'] > 80:
            send_alert('High CPU Usage Alert')
```

## Notas Importantes

1. **Segurança**:
   - Manter as credenciais AWS seguras
   - Revisar security groups regularmente
   - Implementar rotação de chaves

2. **Performance**:
   - Monitorar latência do CloudFront
   - Ajustar auto scaling conforme necessário
   - Otimizar consultas DynamoDB

3. **Custos**:
   - Monitorar uso de recursos
   - Implementar tags para controle de custos
   - Revisar recursos não utilizados

4. **Manutenção**:
   - Backup diário dos dados
   - Atualização regular das dependências
   - Teste regular dos procedimentos de DR
