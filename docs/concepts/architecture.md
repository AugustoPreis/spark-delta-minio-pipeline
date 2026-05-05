# 🏛️ Arquitetura da Pipeline

Entenda como os componentes trabalham juntos.

## Diagrama Geral

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAMADAS DE DADOS                             │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ SOURCE LAYER (Sistema de Origem)                            │
│ ┌────────────────────────────────────────────────────────┐  │
│ │  SQL Server 2025 (LojoDB)                              │  │
│ │  - 11 Tabelas relacionais                              │  │
│ │  - ~3.000+ registros de exemplo                        │  │
│ │  - Dados estruturados e normalizados                   │  │
│ └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              ↓
                    [NOTEBOOK 00: Setup]
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ LANDING ZONE (Zona de Pouso)                                │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ MinIO Bucket: landing-zone/                            │  │
│ │ ├── Formato: CSV (raw, sem transformação)              │  │
│ │ ├── 11 arquivos (um por tabela)                        │  │
│ │ ├── Com headers                                        │  │
│ │ └── Imutável (não modifica)                            │  │
│ └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              ↓
                    [NOTEBOOK 01: Extração]
                    (SQL Server → MinIO)
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ BRONZE LAYER (Layer 1 - Estruturado)                        │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ MinIO Bucket: bronze/                                  │  │
│ │ ├── Formato: Delta Lake (ACID)                         │  │
│ │ ├── 11 tabelas Delta                                   │  │
│ │ ├── Schema enforcement                                 │  │
│ │ ├── Versionamento automático                           │  │
│ │ ├── Time Travel habilitado                             │  │
│ │ └── DML operations (INSERT/UPDATE/DELETE)              │  │
│ └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              ↓
                    [NOTEBOOK 02: Conversão]
                    (CSV → Delta)
                              ↓
                    [NOTEBOOK 03: DML]
                    (INSERT/UPDATE/DELETE/
                    HISTORY/TIME TRAVEL)
                              ↓
```

## Componentes Principais

### 1. Source System (SQL Server)

```
SQL Server 2025
├── Database: LojoDB
├── Schema: dbo
└── Tabelas:
    ├── regiao (5)
    ├── estado (27)
    ├── municipio (100+)
    ├── marca (10)
    ├── modelo (20)
    ├── cliente (1.000)
    ├── endereco (1.000)
    ├── telefone (500)
    ├── carro (500)
    ├── apolice (200)
    └── sinistro (50)
```

**Características**:
- ✅ Normalizado (relacionamentos via FKs)
- ✅ Estruturado (schemas bem definidos)
- ✅ Transacional (ACID nativo)
- ❌ Não escalável para Big Data

### 2. Storage Layer (MinIO)

MinIO implementa S3-API, permitindo:

```
MinIO (localhost:9020)
├── Bucket: landing-zone/
│   ├── Path: s3://landing-zone/regiao/
│   ├── Path: s3://landing-zone/estado/
│   └── ...
├── Bucket: bronze/
│   ├── Path: s3://bronze/regiao/
│   ├── Path: s3://bronze/estado/
│   └── ...
└── Features:
    ├── Versionamento
    ├── Compressão
    ├── Replicação
    └── Lifecycle policies
```

**Protocolo S3**:
```python
s3://bucket_name/path/to/data
# Mapeado para MinIO:
# s3a://bucket_name/path/to/data
# (endpoint: http://localhost:9020)
```

### 3. Processing Engine (Apache Spark)

Spark conecta os componentes:

```
Spark Driver
├── JDBC Driver (SQL Server)
├── S3A Connector (MinIO)
├── Delta Lake Integration
└── Distributes tasks to Executors

Execution:
1. Read from SQL Server (JDBC)
2. Transform in memory
3. Write to MinIO (S3)
4. Apply Delta format
```

### 4. Data Format (Delta Lake)

Delta adiciona camada ACID:

```
Delta Table Structure:
/minio/bronze/cliente/
├── _delta_log/              # Transaction log
│   ├── 00000000000000000000.json
│   ├── 00000000000000000001.json
│   ├── 00000000000000000002.json
│   └── _last_checkpoint
├── part-00000.parquet       # Data files
├── part-00001.parquet
└── ...

Features:
✅ ACID Transactions
✅ Schema Enforcement
✅ Time Travel
✅ DML Operations
```

## Fluxo de Dados Detalhado

### Fase 1: Setup (Notebook 00)

```
1. Verificar conexão SQL Server
   └─→ Connection: pyodbc
       Host: localhost:1433
       Auth: sa / SqlServer@2025!

2. Criar database LojoDB
   └─→ SQL: CREATE DATABASE LojoDB

3. Criar 11 tabelas
   └─→ SQL: CREATE TABLE [schema]
       Define PKs, FKs, tipos

4. Inserir dados de exemplo
   └─→ SQL: INSERT INTO [table] VALUES (...)
       Total: ~3.000+ registros
```

### Fase 2: Extração (Notebook 01)

```
1. Conexão Spark → SQL Server
   └─→ JDBC URL: jdbc:sqlserver://localhost:1433
       Credentials: sa/password
       Database: LojoDB

2. Leitura de tabelas
   └─→ spark.read.jdbc(...)
       Para cada tabela:
       - Query: SELECT * FROM [table]
       - Resultado: Spark DataFrame

3. Validação de dados
   └─→ df.count()      # Contagem de registros
   └─→ df.printSchema()  # Tipos de dados

4. Escrita em CSV
   └─→ spark.write.csv()
       Formato: s3a://landing-zone/[table]/
       Partições: Automáticas
       Headers: Incluído
```

### Fase 3: Conversão (Notebook 02)

```
1. Leitura de CSVs
   └─→ spark.read.csv()
       Source: s3a://landing-zone/[table]/
       Inferência de schema

2. Transformações (opcionais)
   └─→ Validações de tipo
   └─→ Correções de nulos
   └─→ Renomeação de colunas

3. Escrita em Delta
   └─→ df.write.format("delta")
       Destination: s3a://bronze/[table]/
       Mode: overwrite
       
   Resultado:
   - Criação de _delta_log/
   - Compressão para Parquet
   - Metadados criados
```

### Fase 4: DML Operations (Notebook 03)

```
1. INSERT (Adicionar dados)
   └─→ DeltaTable.merge()
       Insere 100 novos clientes

2. UPDATE (Atualizar dados)
   └─→ DeltaTable.update()
       Modifica status de 50 clientes

3. DELETE (Remover dados)
   └─→ DeltaTable.delete()
       Remove 10 clientes

4. HISTORY (Ver histórico)
   └─→ DeltaTable.history()
       Mostra: version, timestamp, operation

5. TIME TRAVEL (Voltar no tempo)
   └─→ read.option("versionAsOf", 0)
       read.option("timestampAsOf", "2025-05-05 10:00:00")
```

## Comunicação entre Componentes

### SQL Server ↔ Spark (via JDBC)

```
Spark Driver
    ↓
JDBC Driver (pyodbc/JDBC)
    ↓ TCP Port 1433
SQL Server
    ↓
DataFrame (In-Memory)
```

Código:
```python
df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:sqlserver://localhost:1433") \
    .option("dbtable", "cliente") \
    .load()
```

### Spark ↔ MinIO (via S3A)

```
Spark Driver
    ↓
S3A Connector (boto3)
    ↓ HTTP/HTTPS Port 9020
MinIO
    ↓
Object Storage
```

Código:
```python
df.write \
    .format("delta") \
    .save("s3a://bronze/cliente/")
```

### Delta Lake (Built-in)

```
Spark Writes CSV
    ↓
Delta Lake Library
    ↓ Cria _delta_log/
    ↓ Comprime para Parquet
    ↓ Armazena em S3
Delta Table Ready
```

## Configurações Importantes

### Spark Session

```python
spark = SparkSession.builder \
    .appName("pipeline") \
    .config("spark.sql.extensions", 
            "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", 
            "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .config("spark.hadoop.fs.s3a.endpoint", 
            "http://localhost:9020") \
    .config("spark.hadoop.fs.s3a.access.key", 
            "minioadmin") \
    .config("spark.hadoop.fs.s3a.secret.key", 
            "minioadmin") \
    .getOrCreate()
```

### JDBC Connection

```python
connection_url = \
    "jdbc:sqlserver://localhost:1433;" \
    "database=LojoDB;" \
    "encrypt=false;" \
    "trustServerCertificate=true;" \
    "loginTimeout=15;"
```

## Escalabilidade

### Atual (Projeto Educacional)

```
Scale: ~3.000 registros
Processing: Local mode
Duration: 20-40 minutos total
Storage: < 1GB
```

### Produção (Conceitual)

```
Scale: Bilhões de registros
Processing: Cluster mode (múltiplos workers)
Duration: Horas/dias
Storage: Terabytes/Petabytes
```

Spark é projetado para ambos! O código permanece igual.
