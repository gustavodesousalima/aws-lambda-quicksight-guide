# Projeto AWS Lambda + QuickSight Dashboard

## Visão Geral
Este projeto cria um pipeline de dados em tempo real que:
1. Executa uma Lambda a cada 30 minutos
2. Consome dados da API OpenWeatherMap
3. Transforma os dados em formato Parquet
4. Armazena no S3
5. Visualiza no QuickSight

## Pré-requisitos

### 1. Ferramentas Necessárias
```bash
# Instalar AWS CLI (Linux/Mac)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Para Windows, baixe o instalador MSI da AWS

# Instalar AWS SAM CLI
pip install aws-sam-cli

# Verificar instalações
aws --version
sam --version
```

### 2. Criar Usuário IAM na AWS
1. Acesse o console AWS → IAM
2. Clique em "Users" → "Create user"
3. Nome do usuário: `lambda-quicksight-user`
4. Selecione "Attach policies directly"
5. Adicione as seguintes políticas:
   - `AWSLambda_FullAccess`
   - `AmazonS3FullAccess`
   - `CloudWatchFullAccess`
   - `IAMFullAccess`
   - `AWSCloudFormationFullAccess`
6. Clique em "Create user"
7. Vá para o usuário criado → "Security credentials"
8. Clique em "Create access key"
9. Escolha "CLI" → "Create access key"
10. **Anote o Access Key ID e Secret Access Key**

### 3. Configurar AWS CLI
```bash
aws configure
# AWS Access Key ID: [cole-seu-access-key-id]
# AWS Secret Access Key: [cole-seu-secret-access-key]
# Default region name: us-east-1
# Default output format: json

# Testar configuração
aws sts get-caller-identity
```

### 4. Obter API Key do OpenWeatherMap
1. Acesse: https://openweathermap.org/api
2. Clique em "Sign Up" (crie conta gratuita)
3. Confirme seu email
4. Vá para "API keys" no seu dashboard
5. **Copie sua API key** (ex: `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`)
6. Aguarde até 2 horas para ativação da key

### 5. Configurar Região AWS
```bash
# Definir região padrão (escolha uma)
export AWS_DEFAULT_REGION=us-east-1

# Verificar regiões disponíveis
aws ec2 describe-regions --output table
```

## Estrutura do Projeto

```
weather-dashboard/
├── template.yaml          # SAM template
├── src/
│   ├── lambda_function.py # Código da Lambda
│   └── requirements.txt   # Dependências Python
├── events/
│   └── event.json        # Evento de teste
└── README.md
```

## Configuração Detalhada do S3

### 1. Criar Bucket S3 via Console (Alternativa Manual)
```bash
# Criar bucket com nome único
aws s3 mb s3://weather-data-bucket-$(date +%s) --region us-east-1

# Ou via console AWS:
```

**Via Console AWS:**
1. Acesse AWS Console → S3
2. Clique em "Create bucket"
3. **Bucket name**: `weather-data-bucket-[seu-nome]-[data]` (ex: `weather-data-bucket-joao-2024`)
4. **Region**: `us-east-1` (mesma do projeto)
5. **Block Public Access**: Deixe marcado (padrão)
6. **Bucket Versioning**: Enable
7. **Default encryption**: Server-side encryption with Amazon S3 managed keys (SSE-S3)
8. Clique em "Create bucket"

### 2. Configurar Estrutura de Pastas no S3
```bash
# Criar estrutura de pastas (opcional, será criada automaticamente)
aws s3api put-object --bucket weather-data-bucket-[seu-nome] --key weather-data/

# Verificar bucket criado
aws s3 ls
```

### 3. Configurar Políticas do Bucket
```bash
# Criar arquivo de política bucket-policy.json
cat > bucket-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowLambdaAccess",
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::BUCKET_NAME/*"
        },
        {
            "Sid": "AllowQuickSightAccess",
            "Effect": "Allow",
            "Principal": {
                "Service": "quicksight.amazonaws.com"
            },
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::BUCKET_NAME",
                "arn:aws:s3:::BUCKET_NAME/*"
            ]
        }
    ]
}
EOF

# Aplicar política (substitua BUCKET_NAME pelo nome real)
# aws s3api put-bucket-policy --bucket weather-data-bucket-[seu-nome] --policy file://bucket-policy.json
```

### 4. Configurar CORS (se necessário)
```bash
# Criar arquivo cors.json
cat > cors.json << 'EOF'
{
    "CORSRules": [
        {
            "AllowedHeaders": ["*"],
            "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
            "AllowedOrigins": ["*"],
            "ExposeHeaders": []
        }
    ]
}
EOF

# Aplicar CORS
# aws s3api put-bucket-cors --bucket weather-data-bucket-[seu-nome] --cors-configuration file://cors.json
```

### 5. Verificar Configuração do S3
```bash
# Listar buckets
aws s3 ls

# Verificar configuração do bucket
aws s3api get-bucket-location --bucket weather-data-bucket-[seu-nome]
aws s3api get-bucket-versioning --bucket weather-data-bucket-[seu-nome]
```

### 1. Política para a Lambda
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::weather-data-bucket-*/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::weather-data-bucket-*"
        }
    ]
}
```

### 2. Política para QuickSight
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::weather-data-bucket-*",
                "arn:aws:s3:::weather-data-bucket-*/*"
            ]
        }
    ]
}
```

## Código da Lambda

### requirements.txt
```
pandas==2.0.3
pyarrow==12.0.1
requests==2.31.0
boto3==1.28.57
```

### lambda_function.py
```python
import json
import boto3
import pandas as pd
import requests
from datetime import datetime, timezone
import os
import io

def lambda_handler(event, context):
    """
    Lambda function que coleta dados meteorológicos e salva em formato Parquet no S3
    """
    
    # Configurações
    API_KEY = os.environ['OPENWEATHER_API_KEY']
    BUCKET_NAME = os.environ['S3_BUCKET_NAME']
    
    # Lista de cidades para coletar dados
    cities = [
        'São Paulo', 'Rio de Janeiro', 'Brasília', 'Salvador', 'Fortaleza',
        'Belo Horizonte', 'Manaus', 'Curitiba', 'Recife', 'Porto Alegre'
    ]
    
    # Coletar dados das cidades
    weather_data = []
    
    for city in cities:
        try:
            # Fazer requisição para a API
            url = f"http://api.openweathermap.org/data/2.5/weather"
            params = {
                'q': f'{city},BR',
                'appid': API_KEY,
                'units': 'metric',
                'lang': 'pt_br'
            }
            
            response = requests.get(url, params=params)
            response.raise_for_status()
            
            data = response.json()
            
            # Extrair dados relevantes
            weather_record = {
                'timestamp': datetime.now(timezone.utc).isoformat(),
                'city': city,
                'country': 'BR',
                'temperature': data['main']['temp'],
                'feels_like': data['main']['feels_like'],
                'humidity': data['main']['humidity'],
                'pressure': data['main']['pressure'],
                'description': data['weather'][0]['description'],
                'wind_speed': data.get('wind', {}).get('speed', 0),
                'wind_direction': data.get('wind', {}).get('deg', 0),
                'cloudiness': data['clouds']['all'],
                'visibility': data.get('visibility', 0) / 1000,  # Converter para km
                'latitude': data['coord']['lat'],
                'longitude': data['coord']['lon']
            }
            
            weather_data.append(weather_record)
            print(f"Dados coletados para {city}: {weather_record['temperature']}°C")
            
        except Exception as e:
            print(f"Erro ao coletar dados para {city}: {str(e)}")
            continue
    
    if not weather_data:
        return {
            'statusCode': 500,
            'body': json.dumps('Nenhum dado foi coletado')
        }
    
    try:
        # Criar DataFrame
        df = pd.DataFrame(weather_data)
        
        # Converter para Parquet
        parquet_buffer = io.BytesIO()
        df.to_parquet(parquet_buffer, engine='pyarrow', index=False)
        parquet_buffer.seek(0)
        
        # Gerar nome do arquivo com timestamp
        timestamp = datetime.now(timezone.utc).strftime('%Y/%m/%d/%H%M%S')
        s3_key = f"weather-data/{timestamp}_weather.parquet"
        
        # Upload para S3
        s3_client = boto3.client('s3')
        s3_client.put_object(
            Bucket=BUCKET_NAME,
            Key=s3_key,
            Body=parquet_buffer.getvalue(),
            ContentType='application/octet-stream'
        )
        
        print(f"Arquivo salvo no S3: s3://{BUCKET_NAME}/{s3_key}")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': f'Dados de {len(weather_data)} cidades salvos com sucesso',
                's3_location': f's3://{BUCKET_NAME}/{s3_key}',
                'records_count': len(weather_data)
            })
        }
        
    except Exception as e:
        print(f"Erro ao processar dados: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps(f'Erro ao processar dados: {str(e)}')
        }
```

## Template SAM (template.yaml)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Weather Data Pipeline with Lambda and QuickSight

Parameters:
  OpenWeatherApiKey:
    Type: String
    NoEcho: true
    Description: API Key do OpenWeatherMap
  
  BucketName:
    Type: String
    Default: weather-data-bucket
    Description: Nome do bucket S3 (será criado com sufixo único)

Globals:
  Function:
    Timeout: 300
    MemorySize: 512
    Runtime: python3.9

Resources:
  # S3 Bucket para dados
  WeatherDataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${BucketName}-${AWS::AccountId}-${AWS::Region}"
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      VersioningConfiguration:
        Status: Enabled

  # Lambda Function
  WeatherDataFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: lambda_function.lambda_handler
      Environment:
        Variables:
          OPENWEATHER_API_KEY: !Ref OpenWeatherApiKey
          S3_BUCKET_NAME: !Ref WeatherDataBucket
      Policies:
        - S3WritePolicy:
            BucketName: !Ref WeatherDataBucket
        - S3ReadPolicy:
            BucketName: !Ref WeatherDataBucket
      Events:
        ScheduleEvent:
          Type: Schedule
          Properties:
            Schedule: rate(30 minutes)
            Description: Executar a cada 30 minutos
            Enabled: true

  # Role para QuickSight
  QuickSightRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "QuickSight-WeatherData-Role-${AWS::Region}"
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: quicksight.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: QuickSightS3Access
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:GetObjectVersion
                  - s3:ListBucket
                Resource:
                  - !Sub "${WeatherDataBucket}/*"
                  - !Ref WeatherDataBucket

Outputs:
  WeatherDataFunction:
    Description: "ARN da Lambda Function"
    Value: !GetAtt WeatherDataFunction.Arn
  
  S3Bucket:
    Description: "Nome do bucket S3"
    Value: !Ref WeatherDataBucket
  
  QuickSightRole:
    Description: "ARN da Role para QuickSight"
    Value: !GetAtt QuickSightRole.Arn
```

## Deploy do Projeto

### 1. Preparar Ambiente Local
```bash
# Criar diretório do projeto
mkdir weather-dashboard
cd weather-dashboard

# Criar estrutura de diretórios
mkdir src events

# Verificar estrutura
ls -la
```

### 2. Criar Arquivos do Projeto

**Crie o arquivo `template.yaml`** (copie todo o conteúdo do Template SAM acima)

**Crie o arquivo `src/requirements.txt`:**
```bash
cat > src/requirements.txt << 'EOF'
pandas==2.0.3
pyarrow==12.0.1
requests==2.31.0
boto3==1.28.57
EOF
```

**Crie o arquivo `src/lambda_function.py`** (copie todo o código Python acima)

**Crie arquivo de teste `events/event.json`:**
```bash
cat > events/event.json << 'EOF'
{
    "test": true
}
EOF
```

### 3. Validar Template SAM
```bash
# Validar sintaxe do template
sam validate

# Se houver erros, corrija o template.yaml
```

### 4. Build do Projeto
```bash
# Build do projeto (primeira vez pode demorar)
sam build

# Verificar se build foi bem-sucedido
ls -la .aws-sam/
```

### 5. Deploy Interativo (Primeira Vez)
```bash
# Deploy com configuração guiada
sam deploy --guided

# Responder as perguntas:
```

**Perguntas do Deploy:**
```
Stack Name [sam-app]: weather-dashboard
AWS Region [us-east-1]: us-east-1
Parameter OpenWeatherApiKey []: [COLE-SUA-API-KEY-AQUI]
Parameter BucketName [weather-data-bucket]: weather-data-bucket-[seu-nome]
Confirm changes before deploy [y/N]: y
Allow SAM CLI to create IAM roles [Y/n]: Y
Disable rollback [y/N]: N
Save parameters to configuration file [Y/n]: Y
SAM configuration file [samconfig.toml]: [ENTER]
SAM configuration environment [default]: [ENTER]
```

### 6. Aguardar Deploy
```bash
# O deploy pode levar 5-10 minutos
# Aguarde até ver:
# Successfully created/updated stack - weather-dashboard in us-east-1
```

### 7. Verificar Deploy
```bash
# Listar stacks do CloudFormation
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE

# Ver recursos criados
aws cloudformation describe-stack-resources --stack-name weather-dashboard

# Testar Lambda
aws lambda list-functions --query 'Functions[?contains(FunctionName, `weather`)].FunctionName'
```

### 8. Deploy Subsequentes (Após Mudanças)
```bash
# Após modificar código ou template
sam build && sam deploy

# Ou se quiser confirmar mudanças
sam build && sam deploy --confirm-changeset
```

### 9. Testar Lambda Manualmente
```bash
# Obter nome da função
FUNCTION_NAME=$(aws lambda list-functions --query 'Functions[?contains(FunctionName, `WeatherData`)].FunctionName' --output text)

# Invocar função
aws lambda invoke \
    --function-name $FUNCTION_NAME \
    --payload '{}' \
    --cli-binary-format raw-in-base64-out \
    response.json

# Ver resposta
cat response.json
```

### 10. Verificar Dados no S3
```bash
# Listar arquivos criados
aws s3 ls s3://weather-data-bucket-[seu-nome]/weather-data/ --recursive

# Baixar um arquivo para testar
aws s3 cp s3://weather-data-bucket-[seu-nome]/weather-data/2024/01/15/120000_weather.parquet ./teste.parquet
```

## Configuração Completa do QuickSight

### 1. Ativar QuickSight na AWS
```bash
# Verificar se QuickSight está disponível na região
aws quicksight list-users --aws-account-id $(aws sts get-caller-identity --query Account --output text)
```

**Via Console AWS:**
1. Acesse AWS Console → QuickSight
2. Se não estiver ativo, clique em "Sign up for QuickSight"
3. Escolha **"Standard Edition"** (gratuito por 30 dias)
4. Preencha informações:
   - **QuickSight account name**: `weather-dashboard`
   - **Notification email**: seu-email@exemplo.com
5. Marque as opções:
   - ✅ Amazon S3
   - ✅ Amazon Athena
6. Clique em "Finish"

### 2. Configurar Permissões para S3
1. No QuickSight, vá para o ícone do usuário (canto superior direito)
2. Clique em "Manage QuickSight"
3. Vá para "Security & permissions"
4. Clique em "Add or remove"
5. Marque **Amazon S3**
6. Clique em "Select S3 buckets"
7. Encontre e marque seu bucket: `weather-data-bucket-[seu-nome]`
8. Clique em "Select buckets" → "Update"

### 3. Criar Data Source
1. No QuickSight, clique em **"Datasets"** (menu lateral)
2. Clique em **"New dataset"**
3. Escolha **"S3"**
4. Configure o Data Source:
   - **Data source name**: `Weather Data Source`
   - **S3 bucket URL**: `s3://weather-data-bucket-[seu-nome]/weather-data/`
   - **S3 bucket region**: `us-east-1`
5. Clique em **"Connect"**

### 4. Aguardar Dados e Configurar Dataset
```bash
# Primeiro, certifique-se de que há dados no S3
aws s3 ls s3://weather-data-bucket-[seu-nome]/weather-data/ --recursive

# Se não houver dados, execute a Lambda manualmente
FUNCTION_NAME=$(aws lambda list-functions --query 'Functions[?contains(FunctionName, `WeatherData`)].FunctionName' --output text)
aws lambda invoke --function-name $FUNCTION_NAME --payload '{}' response.json
```

**Configurar Dataset:**
1. Após conectar ao S3, o QuickSight detectará os arquivos Parquet
2. Selecione o arquivo mais recente
3. Escolha **"Import to SPICE for quicker analytics"**
4. Clique em **"Edit/Preview data"**
5. Verifique se os tipos de dados estão corretos:
   - `timestamp`: DateTime
   - `temperature`: Decimal
   - `humidity`: Integer
   - `pressure`: Decimal
   - `city`: String
6. Clique em **"Save & publish"**
7. Nome do dataset: `Weather Dataset`

### 5. Criar Analysis e Dashboard
1. Após salvar o dataset, clique em **"Create analysis"**
2. Será aberto o editor de visualizações

**Adicionar Visualizações:**

**A. Gráfico de Temperatura por Tempo:**
1. Clique em **"Add visual"**
2. Escolha **"Line chart"**
3. Arraste `timestamp` para **X-axis**
4. Arraste `temperature` para **Value**
5. Arraste `city` para **Color**
6. Título: "Temperatura por Tempo"

**B. Mapa de Temperaturas:**
1. Clique em **"Add visual"**
2. Escolha **"Geospatial chart"**
3. Arraste `latitude` para **Latitude**
4. Arraste `longitude` para **Longitude**
5. Arraste `temperature` para **Color**
6. Arraste `city` para **Label**
7. Título: "Mapa de Temperaturas"

**C. Gráfico de Barras - Umidade:**
1. Clique em **"Add visual"**
2. Escolha **"Vertical bar chart"**
3. Arraste `city` para **X-axis**
4. Arraste `humidity` para **Value**
5. Título: "Umidade por Cidade"

**D. KPIs:**
1. Clique em **"Add visual"**
2. Escolha **"KPI"**
3. Arraste `temperature` para **Value**
4. Configure aggregation: **Average**
5. Título: "Temperatura Média"

### 6. Configurar Refresh Automático
1. Vá para **"Datasets"** no menu lateral
2. Clique no dataset **"Weather Dataset"**
3. Clique em **"Schedule refresh"**
4. Clique em **"Create schedule"**
5. Configure:
   - **Refresh type**: Full refresh
   - **Time zone**: America/Sao_Paulo
   - **Repeats**: Daily
   - **Starting time**: A cada 30 minutos a partir de agora
6. Clique em **"Create"**

### 7. Publicar Dashboard
1. Na analysis, clique em **"Share"** → **"Publish dashboard"**
2. Nome do dashboard: `Weather Dashboard`
3. Clique em **"Publish dashboard"**
4. Clique em **"Save & publish"**

### 8. Configurar Alertas (Opcional)
1. No dashboard, clique em uma visualização
2. Clique em **"..."** → **"Create alert"**
3. Configure alertas para:
   - Temperatura > 35°C
   - Umidade < 30%
4. Adicione seu email para notificações

### 9. Testar Refresh Manual
1. Vá para **"Datasets"**
2. Clique no dataset **"Weather Dataset"**
3. Clique em **"Refresh now"**
4. Aguarde a atualização
5. Volte ao dashboard para ver dados atualizados

## Teste e Monitoramento Completo

### 1. Verificar Stack do CloudFormation
```bash
# Listar stacks
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE

# Ver detalhes do stack
aws cloudformation describe-stacks --stack-name weather-dashboard

# Ver recursos criados
aws cloudformation describe-stack-resources --stack-name weather-dashboard --output table
```

### 2. Testar Lambda Detalhadamente
```bash
# Obter informações da função
FUNCTION_NAME=$(aws cloudformation describe-stack-resources \
    --stack-name weather-dashboard \
    --query 'StackResources[?ResourceType==`AWS::Lambda::Function`].PhysicalResourceId' \
    --output text)

echo "Nome da função: $FUNCTION_NAME"

# Testar função
aws lambda invoke \
    --function-name $FUNCTION_NAME \
    --payload '{}' \
    --cli-binary-format raw-in-base64-out \
    response.json

# Ver resposta
echo "Resposta da Lambda:"
cat response.json | python -m json.tool

# Ver logs da execução
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/$FUNCTION_NAME"
```

### 3. Monitorar Logs em Tempo Real
```bash
# Instalar ferramenta para logs (se não tiver)
pip install awslogs

# Ver logs em tempo real
sam logs -n WeatherDataFunction --stack-name weather-dashboard --tail

# Ou usar AWS CLI
aws logs tail /aws/lambda/$FUNCTION_NAME --follow

# Ver logs de execuções específicas
aws logs filter-log-events \
    --log-group-name "/aws/lambda/$FUNCTION_NAME" \
    --start-time $(date -d '1 hour ago' +%s)000
```

### 4. Verificar S3 Detalhadamente
```bash
# Obter nome do bucket
BUCKET_NAME=$(aws cloudformation describe-stack-resources \
    --stack-name weather-dashboard \
    --query 'StackResources[?ResourceType==`AWS::S3::Bucket`].PhysicalResourceId' \
    --output text)

echo "Nome do bucket: $BUCKET_NAME"

# Listar todos os arquivos
aws s3 ls s3://$BUCKET_NAME/weather-data/ --recursive --human-readable

# Ver último arquivo criado
LATEST_FILE=$(aws s3 ls s3://$BUCKET_NAME/weather-data/ --recursive | sort | tail -n 1 | awk '{print $4}')
echo "Último arquivo: $LATEST_FILE"

# Baixar e verificar conteúdo (requer pandas)
aws s3 cp s3://$BUCKET_NAME/$LATEST_FILE ./latest_weather.parquet

# Verificar conteúdo com Python
python3 << EOF
import pandas as pd
try:
    df = pd.read_parquet('./latest_weather.parquet')
    print("Estrutura dos dados:")
    print(df.info())
    print("\nPrimeiras linhas:")
    print(df.head())
    print(f"\nTotal de registros: {len(df)}")
except Exception as e:
    print(f"Erro ao ler arquivo: {e}")
EOF
```

### 5. Verificar Agendamento EventBridge
```bash
# Listar regras do EventBridge
aws events list-rules --name-prefix "weather-dashboard"

# Ver detalhes da regra de agendamento
RULE_NAME=$(aws events list-rules --query 'Rules[?contains(Name, `weather-dashboard`)].Name' --output text)
aws events describe-rule --name $RULE_NAME

# Ver targets da regra
aws events list-targets-by-rule --rule $RULE_NAME
```

### 6. Monitorar Execuções
```bash
# Ver métricas da Lambda no CloudWatch
aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name Invocations \
    --dimensions Name=FunctionName,Value=$FUNCTION_NAME \
    --statistics Sum \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 3600

# Ver erros
aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name Errors \
    --dimensions Name=FunctionName,Value=$FUNCTION_NAME \
    --statistics Sum \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 3600
```

### 7. Testar QuickSight via API
```bash
# Listar datasets
aws quicksight list-data-sets --aws-account-id $(aws sts get-caller-identity --query Account --output text)

# Ver status de refresh
DATASET_ID=$(aws quicksight list-data-sets \
    --aws-account-id $(aws sts get-caller-identity --query Account --output text) \
    --query 'DataSetSummaries[?contains(Name, `Weather`)].DataSetId' \
    --output text)

if [ ! -z "$DATASET_ID" ]; then
    aws quicksight describe-data-set \
        --aws-account-id $(aws sts get-caller-identity --query Account --output text) \
        --data-set-id $DATASET_ID
fi
```

### 8. Criar Script de Monitoramento
```bash
cat > monitor.sh << 'EOF'
#!/bin/bash

echo "=== MONITORAMENTO WEATHER DASHBOARD ==="
echo "Timestamp: $(date)"
echo

# Obter informações básicas
STACK_NAME="weather-dashboard"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Verificar stack
echo "1. Status do CloudFormation Stack:"
aws cloudformation describe-stacks --stack-name $STACK_NAME --query 'Stacks[0].StackStatus' --output text
echo

# Verificar Lambda
echo "2. Status da Lambda:"
FUNCTION_NAME=$(aws cloudformation describe-stack-resources \
    --stack-name $STACK_NAME \
    --query 'StackResources[?ResourceType==`AWS::Lambda::Function`].PhysicalResourceId' \
    --output text)
aws lambda get-function --function-name $FUNCTION_NAME --query 'Configuration.State' --output text
echo

# Verificar S3
echo "3. Arquivos no S3 (últimos 5):"
BUCKET_NAME=$(aws cloudformation describe-stack-resources \
    --stack-name $STACK_NAME \
    --query 'StackResources[?ResourceType==`AWS::S3::Bucket`].PhysicalResourceId' \
    --output text)
aws s3 ls s3://$BUCKET_NAME/weather-data/ --recursive | tail -5
echo

# Verificar última execução
echo "4. Última execução da Lambda:"
aws logs filter-log-events \
    --log-group-name "/aws/lambda/$FUNCTION_NAME" \
    --start-time $(date -d '2 hours ago' +%s)000 \
    --filter-pattern "\"statusCode\": 200" \
    --query 'events[-1].message' \
    --output text
echo

echo "=== FIM DO MONITORAMENTO ==="
EOF

chmod +x monitor.sh

# Executar monitoramento
./monitor.sh
```

### 9. Criar Alarmes no CloudWatch
```bash
# Criar alarme para falhas na Lambda
aws cloudwatch put-metric-alarm \
    --alarm-name "WeatherDashboard-Lambda-Errors" \
    --alarm-description "Alarme para erros na Lambda" \
    --metric-name Errors \
    --namespace AWS/Lambda \
    --statistic Sum \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --dimensions Name=FunctionName,Value=$FUNCTION_NAME \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:$ACCOUNT_ID:weather-alerts

# Criar alarme para duração da Lambda
aws cloudwatch put-metric-alarm \
    --alarm-name "WeatherDashboard-Lambda-Duration" \
    --alarm-description "Alarme para duração alta da Lambda" \
    --metric-name Duration \
    --namespace AWS/Lambda \
    --statistic Average \
    --period 300 \
    --threshold 60000 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=FunctionName,Value=$FUNCTION_NAME \
    --evaluation-periods 2
```

### 10. Solução de Problemas Comuns

**Problema: Lambda retorna erro 500**
```bash
# Ver logs detalhados
aws logs filter-log-events \
    --log-group-name "/aws/lambda/$FUNCTION_NAME" \
    --filter-pattern "ERROR"

# Verificar variáveis de ambiente
aws lambda get-function-configuration --function-name $FUNCTION_NAME --query 'Environment'
```

**Problema: Dados não aparecem no QuickSight**
```bash
# Verificar se arquivos estão sendo criados
aws s3 ls s3://$BUCKET_NAME/weather-data/ --recursive

# Forçar refresh do dataset
aws quicksight create-refresh \
    --aws-account-id $ACCOUNT_ID \
    --data-set-id $DATASET_ID
```

**Problema: API Key inválida**
```bash
# Testar API key manualmente
curl "http://api.openweathermap.org/data/2.5/weather?q=São Paulo,BR&appid=SUA_API_KEY"

# Atualizar variável de ambiente da Lambda
aws lambda update-function-configuration \
    --function-name $FUNCTION_NAME \
    --environment Variables="{OPENWEATHER_API_KEY=nova-api-key,S3_BUCKET_NAME=$BUCKET_NAME}"
```
# Ver logs da Lambda
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/weather-dashboard

# Ver logs em tempo real
sam logs -n WeatherDataFunction --stack-name weather-dashboard --tail
```

### 3. Verificar S3
```bash
# Listar arquivos no bucket
aws s3 ls s3://weather-data-bucket-[account-id]-[region]/weather-data/ --recursive
```

## Estrutura dos Dados no S3

```
weather-data-bucket-[account-id]-[region]/
└── weather-data/
    ├── 2024/01/15/
    │   ├── 120000_weather.parquet
    │   ├── 123000_weather.parquet
    │   └── 130000_weather.parquet
    └── 2024/01/16/
        ├── 120000_weather.parquet
        └── 123000_weather.parquet
```

## Custos Estimados (mensal)

- **Lambda**: ~$0.20 (1440 execuções/mês)
- **S3**: ~$1.00 (1GB dados)
- **QuickSight**: $9.00 (Standard Edition)
- **CloudWatch**: ~$0.50 (logs)
- **Total**: ~$10.70/mês

## Troubleshooting

### Erro na Lambda
```bash
# Ver logs detalhados
sam logs -n WeatherDataFunction --stack-name weather-dashboard --tail

# Testar localmente
sam local invoke WeatherDataFunction --event events/event.json
```

### Erro no QuickSight
1. Verificar permissões da role
2. Confirmar formato dos arquivos Parquet
3. Verificar estrutura do S3

### Atualizar código
```bash
# Após modificar o código
sam build && sam deploy
```

## Próximos Passos

1. **Adicionar mais APIs**: Integrar outras fontes de dados
2. **Alertas**: Configurar CloudWatch Alarms
3. **Backup**: Implementar retenção de dados
4. **Segurança**: Adicionar criptografia
5. **Performance**: Otimizar consultas no QuickSight

## Recursos Úteis

- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
- [QuickSight User Guide](https://docs.aws.amazon.com/quicksight/)
- [OpenWeatherMap API](https://openweathermap.org/api)
- [Pandas to Parquet](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_parquet.html)
