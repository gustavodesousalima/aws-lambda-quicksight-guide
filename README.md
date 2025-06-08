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
# Instalar AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Instalar AWS SAM CLI
pip install aws-sam-cli

# Verificar instalações
aws --version
sam --version
```

### 2. Configurar AWS CLI
```bash
aws configure
# AWS Access Key ID: [sua-access-key]
# AWS Secret Access Key: [sua-secret-key]
# Default region name: us-east-1
# Default output format: json
```

### 3. Obter API Key do OpenWeatherMap
1. Acesse: https://openweathermap.org/api
2. Crie uma conta gratuita
3. Obtenha sua API key

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

## Configuração do IAM

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

### 1. Criar estrutura de arquivos
```bash
mkdir weather-dashboard
cd weather-dashboard

# Criar diretórios
mkdir src events

# Criar arquivos (copie o conteúdo acima)
# template.yaml, src/lambda_function.py, src/requirements.txt
```

### 2. Build e Deploy
```bash
# Build do projeto
sam build

# Deploy interativo (primeira vez)
sam deploy --guided

# Responder as perguntas:
# Stack Name: weather-dashboard
# AWS Region: us-east-1
# Parameter OpenWeatherApiKey: [sua-api-key]
# Parameter BucketName: weather-data-bucket
# Confirm changes before deploy: Y
# Allow SAM CLI to create IAM roles: Y
# Save parameters to configuration file: Y
```

### 3. Deploy subsequentes
```bash
sam build && sam deploy
```

## Configuração do QuickSight

### 1. Ativar QuickSight
1. Acesse o console da AWS
2. Vá para QuickSight
3. Clique em "Sign up for QuickSight" se necessário
4. Escolha "Standard Edition"
5. Configure sua conta

### 2. Criar Data Source
1. No QuickSight, clique em "Datasets"
2. Clique em "New dataset"
3. Escolha "S3"
4. Configure:
   - **Data source name**: Weather Data
   - **S3 bucket**: weather-data-bucket-[account-id]-[region]
   - **URL**: s3://weather-data-bucket-[account-id]-[region]/weather-data/
5. Clique em "Connect"

### 3. Preparar dados
1. Escolha "Import to SPICE for quicker analytics"
2. Clique em "Edit/Preview data"
3. Configure os tipos de dados se necessário
4. Clique em "Save & publish"

### 4. Criar Dashboard
1. Clique em "Create analysis"
2. Adicione visualizações:
   - **Gráfico de linha**: Temperatura por tempo
   - **Mapa**: Temperatura por cidade
   - **Gráfico de barras**: Umidade por cidade
   - **KPI**: Temperatura média

### 5. Configurar Refresh Automático
1. No dataset, clique em "Schedule refresh"
2. Configure para executar a cada 30 minutos
3. Salve a configuração

## Teste e Monitoramento

### 1. Testar Lambda manualmente
```bash
# Invocar a função
aws lambda invoke \
    --function-name weather-dashboard-WeatherDataFunction-[ID] \
    --payload '{}' \
    response.json

# Ver resposta
cat response.json
```

### 2. Verificar logs
```bash
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
