# 🪣 MinIO - Object Storage

Guia prático para MinIO neste projeto.

## O que é MinIO?

MinIO é uma implementação de Object Storage S3-compatível, leve e de código aberto.

```
MinIO ≈ AWS S3 (local)

AWS S3 (Cloud):           MinIO (Local):
- Gerenciado AWS          - Seu servidor
- Global                  - Private/On-prem
- Caro (pay per use)      - Barato
- Escalável infinito      - Escalável conforme hardware
- Durabilidade garantida  - Você gerencia backup
```

## Arquitetura

### Conceitos

```
MinIO
├── Buckets (pastas de topo)
│   ├── landing-zone/
│   │   ├── regiao/
│   │   ├── estado/
│   │   └── ...
│   └── bronze/
│       ├── regiao/
│       ├── estado/
│       └── ...
└── Objetos (arquivos)
    ├── regiao.csv
    ├── data.json
    └── table.parquet
```

### S3 API

MinIO implementa a API S3 da AWS:

```python
# Usando boto3 (cliente S3)
import boto3

client = boto3.client(
    's3',
    endpoint_url='http://localhost:9020',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin'
)

# Listar buckets
response = client.list_buckets()
print(response['Buckets'])

# Criar bucket
client.create_bucket(Bucket='meu-bucket')

# Upload
client.upload_file('local.csv', 'meu-bucket', 'remoto.csv')

# Download
client.download_file('meu-bucket', 'remoto.csv', 'local.csv')

# Listar objetos
response = client.list_objects_v2(Bucket='meu-bucket')
for obj in response['Contents']:
    print(obj['Key'])

# Delete
client.delete_object(Bucket='meu-bucket', Key='arquivo.csv')
```

## Acesso via Spark

### S3A Connector

```python
spark = SparkSession.builder \
    .appName("MinIO") \
    .config("spark.hadoop.fs.s3a.endpoint", "http://localhost:9020") \
    .config("spark.hadoop.fs.s3a.access.key", "minioadmin") \
    .config("spark.hadoop.fs.s3a.secret.key", "minioadmin") \
    .config("spark.hadoop.fs.s3a.path.style.access", "true") \
    .config("spark.hadoop.fs.s3a.impl", 
            "org.apache.hadoop.fs.s3a.S3AFileSystem") \
    .getOrCreate()

# Agora pode ler/escrever em S3
df = spark.read.csv("s3a://landing-zone/cliente.csv")
df.write.parquet("s3a://bronze/cliente/")
```

### Paths S3A

```python
# Path format
"s3a://bucket_name/path/to/object"

# Exemplos
"s3a://landing-zone/regiao/"
"s3a://landing-zone/regiao/part-00000.csv"
"s3a://bronze/cliente/_delta_log/00000000000000000000.json"
```

## Console Web

Acessar http://localhost:9021

```
Login:
- Usuário: minioadmin
- Senha: minioadmin

Interface:
- Buckets na esquerda
- Objetos no centro
- Upload/Download botões
```

### Operações no Console

1. **Ver Buckets**: Esquerda mostra lista
2. **Entrar em Bucket**: Clique nome
3. **Upload Arquivo**: Botão "Upload"
4. **Download Arquivo**: Clique arquivo → Download
5. **Deletar**: Clique arquivo → Delete
6. **Criar Bucket**: Botão "Create Bucket"

## Gerenciamento

### Via AWS CLI

```bash
# Instalar
pip install awscli

# Configurar
aws configure set aws_access_key_id minioadmin
aws configure set aws_secret_access_key minioadmin
aws configure set region us-east-1

# Usar com MinIO
alias minio-aws='aws --endpoint-url http://localhost:9020'

# Listar buckets
minio-aws s3 ls

# Listar objetos
minio-aws s3 ls s3://landing-zone/

# Copiar
minio-aws s3 cp arquivo.csv s3://landing-zone/

# Sincronizar diretório
minio-aws s3 sync ./data/ s3://landing-zone/
```

### Via Docker

```bash
# Acessar container MinIO
docker exec -it minio /bin/bash

# Usar mc (MinIO Client)
mc alias set local http://localhost:9020 minioadmin minioadmin
mc ls local
mc cat local/landing-zone/regiao.csv
```

## Buckets neste Projeto

### landing-zone

```
Propósito: Raw data (zona de pouso)
Formato: CSV
Modo: Append-only
Conteúdo:
├── regiao/
├── estado/
├── municipio/
├── marca/
├── modelo/
├── cliente/
├── endereco/
├── telefone/
├── carro/
├── apolice/
└── sinistro/
```

### bronze

```
Propósito: Delta Lake (dados estruturados)
Formato: Delta (Parquet + JSON metadata)
Modo: Transacional
Conteúdo:
├── regiao/
│   ├── _delta_log/
│   ├── part-00000.parquet
│   └── ...
├── estado/
├── ...
```

## Performance

### Multipart Upload

```python
# Para arquivos grandes
client.upload_file(
    'arquivo_grande.zip',
    'bucket',
    'destino.zip',
    Config=TransferConfig(
        multipart_threshold=1024*25,  # 25MB
        max_concurrency=10
    )
)
```

### Particionamento

Spark automaticamente particiona:

```python
# Spark escreve múltiplos arquivos
df.write.parquet("s3a://bucket/path/")

# Resultado:
# s3://bucket/path/
# ├── part-00000.parquet
# ├── part-00001.parquet
# ├── part-00002.parquet
# └── _SUCCESS
```

## Monitoramento

### Verificar Espaço

```bash
# Via Docker
docker exec minio df -h /export/

# Via console
- Menu → Settings → Metrics
```

### Logs

```bash
# Docker logs
docker compose logs minio

# Arquivo (se configurado)
# /export/logs/
```

## Problemas Comuns

### Erro: "Access Denied"

```python
# Verificar credenciais
spark = SparkSession.builder \
    .config("spark.hadoop.fs.s3a.access.key", "minioadmin") \
    .config("spark.hadoop.fs.s3a.secret.key", "minioadmin") \
    .getOrCreate()
```

### Erro: "Bucket not found"

```python
# Criar bucket se não existir
import boto3

client = boto3.client(
    's3',
    endpoint_url='http://localhost:9020',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin'
)

try:
    client.create_bucket(Bucket='landing-zone')
except client.exceptions.BucketAlreadyExists:
    pass
```

### Erro: "Connection refused"

```bash
# Verificar se MinIO está rodando
docker compose ps minio

# Se não, iniciar
docker compose up -d minio

# Aguardar 10 segundos
sleep 10
```

## Integração com Spark

### Leitura

```python
# CSV de MinIO
df = spark.read \
    .option("header", "true") \
    .csv("s3a://landing-zone/cliente/")

# JSON
df = spark.read.json("s3a://landing-zone/dados.json")

# Delta
df = spark.read.format("delta") \
    .load("s3a://bronze/cliente/")

# Parquet
df = spark.read.parquet("s3a://archive/cliente/")
```

### Escrita

```python
# Para CSV
df.write.csv("s3a://landing-zone/novo/")

# Para Delta
df.write \
    .format("delta") \
    .mode("overwrite") \
    .save("s3a://bronze/novo/")

# Particionado
df.write \
    .format("delta") \
    .partitionBy("ano", "mes") \
    .mode("overwrite") \
    .save("s3a://bronze/vendas/")
```
