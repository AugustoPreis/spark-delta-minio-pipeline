# Spark Delta Lake + MinIO + SQL Server

> Uma solução end-to-end de engenharia de dados demonstrando a extração, armazenamento e processamento de dados usando Apache Spark, Delta Lake, MinIO e SQL Server em uma arquitetura moderna de lakehouse.

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![PySpark 3.5.3](https://img.shields.io/badge/pyspark-3.5.3-orange.svg)](https://spark.apache.org/)
[![Delta Lake 3.2.0](https://img.shields.io/badge/delta%20lake-3.2.0-green.svg)](https://delta.io/)
[![Docker Compose](https://img.shields.io/badge/docker%20compose-v2+-blue.svg)](https://docs.docker.com/compose/)

## 📋 Visão Geral

Este projeto implementa uma pipeline de dados moderna seguindo a **arquitetura medalhão** (landing zone → bronze layer), demonstrando:

1. **Extração**: Leitura de dados de um banco SQL Server em produção
2. **Armazenamento Intermediário**: Persistência em formato CSV no MinIO (landing zone)
3. **Transformação**: Conversão para Delta Lake com garantias ACID
4. **Manipulação de Dados**: Operações DML completas (INSERT, UPDATE, DELETE)
5. **Versionamento**: Histórico e Time Travel nos dados

### Use Cases

- 📚 Aprendizado de Spark + Delta Lake em ambiente controlado
- 🏗️ Prototipagem de arquiteturas lakehouse
- 📊 Demonstração de processamento de dados em larga escala
- 🔄 Exemplo de integração SQL Server → Cloud/Object Storage

---

## 🏛️ Arquitetura

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          PIPELINE DE DADOS                               │
└──────────────────────────────────────────────────────────────────────────┘

        Notebook 00              Notebook 01            Notebooks 02-03
     ┌──────────────┐         ┌──────────────┐        ┌──────────────────┐
     │ Setup SQLServer         │ Extração     │        │ Transformação    │
     │              │         │              │        │ + DML            │
     ▼              ▼         ▼              ▼        ▼                   ▼
┌─────────────┐ ┌──────────────────────┐ ┌────────────────────────────────┐
│ SQL Server  │ │ MinIO (S3-compat)    │ │ Delta Lake (Bronze)            │
│   LojoDB    │ │ landing-zone/        │ │ /minio/bronze/                 │
│             │ │                      │ │                                │
│  11 Tabelas │─▶│  11 CSVs             │─▶│  11 Tabelas Delta             │
│  Dados de   │ │  (Zona de Pouso)     │ │  (ACID + History)              │
│  Exemplo    │ │                      │ │                                │
└─────────────┘ └──────────────────────┘ │  INSERT/UPDATE/DELETE          │
                                         │  + Time Travel                 │
                                         └────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                     CAMADAS DE ARMAZENAMENTO                              │
├──────────────────────────────────────────────────────────────────────────┤
│ Landing Zone:  Raw data extraído (CSV)        → landing-zone/            │
│ Bronze Layer:  Delta tables processadas       → bronze/                  │
└──────────────────────────────────────────────────────────────────────────┘
```

### Fluxo de Dados

```
SQL Server
    ↓
[ODBC / pyodbc]
    ↓
Spark DataFrame
    ↓
[MinIO S3 API]
    ↓
landing-zone (CSV)
    ↓
[PySpark CSV Reader]
    ↓
Spark DataFrame
    ↓
[Delta Writer]
    ↓
bronze (Delta Table)
    ↓
[DML Operations]
    ↓
Delta Table with History/Time Travel
```

---

## 🔧 Pré-requisitos

### Sistema Operacional

- **Linux** (Ubuntu 20.04+, Debian 11+, ou WSL2 no Windows 11)
- **macOS** 10.15+ (com homebrew)

> Windows puro não é suportado; use WSL2.

### Ferramentas Necessárias

| Ferramenta | Versão | Propósito |
|-----------|--------|----------|
| Docker Engine | 20.10+ | Containerização |
| Docker Compose | v2.0+ | Orquestração |
| Python | 3.11.x | Interpretador |
| Java | OpenJDK 11+ | Runtime Spark |
| UV | Latest | Gerenciador de pacotes |
| ODBC Driver 18 | Latest | Conector SQL Server |

### Dependências Python

Automaticamente instaladas via `uv sync`:

```
apache-spark==3.5.3
delta-spark==3.2.0
boto3>=1.34.0
pyodbc>=5.1.0
pandas>=2.2.0
jupyterlab>=4.2.5
python-dotenv>=1.0.0
mkdocs>=1.5.0
mkdocs-material>=9.4.0
```

---

## 🚀 Quick Start

### 1. Clonar o Repositório

```bash
git clone https://github.com/AugustoPreis/spark-delta-minio-sqlserver.git
cd spark-delta-minio-sqlserver
```

### 2. Subir Infraestrutura

```bash
# Inicia SQL Server 2025 + MinIO
docker compose up -d

# Verificar se está tudo funcionando
docker compose ps
```

Espere 30-60 segundos para SQL Server estar pronto.

### 3. Configurar Ambiente Python

```bash
# Criar ambiente virtual com UV
uv venv

# Ativar
source .venv/bin/activate

# Instalar dependências
uv sync
```

### 4. Configurar Variáveis de Ambiente

```bash
# Copiar template
cp .env.example .env

# Editar .env se necessário (padrões já pré-configurados)
nano .env
```

### 5. Configurar Driver ODBC (Ubuntu/Debian)

```bash
# Adicionar repositório Microsoft
curl https://packages.microsoft.com/keys/microsoft.asc | sudo tee /etc/apt/trusted.gpg.d/microsoft.asc
sudo curl https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/prod.list | \
  sudo tee /etc/apt/sources.list.d/mssql-release.list

# Instalar
sudo apt update
sudo ACCEPT_EULA=Y apt install -y msodbcsql18 unixodbc-dev

# Validar
odbcinst -q -d  # Deve retornar: [ODBC Driver 18 for SQL Server]
```

**macOS (Homebrew):**
```bash
brew tap microsoft/mssql-release https://github.com/Microsoft/homebrew-mssql-release
brew install msodbcsql18 mssql-tools18
```

### 6. Iniciar Jupyter Lab

```bash
jupyter lab
```

> ⚠️ **IMPORTANTE**: Selecione o kernel `.venv` antes de executar os notebooks.

---

## 📔 Notebooks

Execute **nesta ordem**:

### 00_setup_sqlserver.ipynb

**Objetivo**: Criar database e carregar dados iniciais

**Ações**:
- ✅ Cria database `LojoDB` no SQL Server
- ✅ Cria 11 tabelas com estrutura de exemplo
- ✅ Insere dados de exemplo em todas as tabelas

**Tabelas Criadas**:
| Tabela | Registros | Descrição |
|--------|-----------|-----------|
| regiao | 5 | Regiões geográficas (Norte, Nordeste, etc.) |
| estado | 27 | Estados brasileiros |
| municipio | 100+ | Municípios |
| marca | 10 | Marcas de veículos (Fiat, Ford, Volkswagen...) |
| modelo | 20 | Modelos de veículos |
| cliente | 1.000 | Clientes da loja |
| endereco | 1.000 | Endereços dos clientes |
| telefone | 500 | Telefones dos clientes |
| carro | 500 | Carros do inventário |
| apolice | 200 | Apólices de seguro |
| sinistro | 50 | Sinistros registrados |

**Tempo Esperado**: 5-10 minutos

### 01_sqlserver_to_minio_csv.ipynb

**Objetivo**: Extrair dados do SQL Server para MinIO em formato CSV

**Ações**:
- ✅ Conecta ao SQL Server com `pyodbc`
- ✅ Lê todas as 11 tabelas
- ✅ Escreve em formato CSV no MinIO bucket `landing-zone`
- ✅ Valida quantidade de registros

**Saída**:
```
minio/landing-zone/
├── regiao.csv
├── estado.csv
├── municipio.csv
├── marca.csv
├── modelo.csv
├── cliente.csv
├── endereco.csv
├── telefone.csv
├── carro.csv
├── apolice.csv
└── sinistro.csv
```

**Tempo Esperado**: 2-5 minutos

### 02_csv_to_delta.ipynb

**Objetivo**: Converter CSVs em tabelas Delta Lake com garantias ACID

**Ações**:
- ✅ Lê CSVs do MinIO
- ✅ Valida schemas e tipos de dados
- ✅ Escreve em formato Delta no bucket `bronze`
- ✅ Cria metadados Delta (histórico, etc.)

**Saída**:
```
minio/bronze/
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

**Recursos Delta**:
- ✨ Versionamento automático
- ✨ ACID transactions
- ✨ Rollback support

**Tempo Esperado**: 3-8 minutos

### 03_dml_delta.ipynb

**Objetivo**: Demonstrar operações DML em Delta Lake

**Operações DML**:

#### INSERT
```python
# Inserir novos registros
delta_table.insert(new_records_df)
```

#### UPDATE
```python
# Atualizar registros existentes
delta_table.update(
    condition = col("id") == 123,
    set = {"status": "INATIVO"}
)
```

#### DELETE
```python
# Deletar registros
delta_table.delete(col("status") == "CANCELADO")
```

#### HISTORY
```python
# Ver histórico de mudanças
spark.sql("SELECT * FROM delta.`/minio/bronze/cliente/` VERSION AS OF 0")
```

#### TIME TRAVEL
```python
# Viajar no tempo
spark.sql("""
  SELECT * FROM delta.`/minio/bronze/cliente/`
  TIMESTAMP AS OF '2025-05-05 10:30:00'
""")
```

**Demonstrações Incluídas**:
- ✅ Insert de 100 novos clientes
- ✅ Update de 50 clientes (status)
- ✅ Delete de 10 clientes (inativos)
- ✅ Visualização de histórico completo
- ✅ Time Travel a versões anteriores

**Tempo Esperado**: 5-10 minutos

---

## 📊 Estrutura do Projeto

```
spark-delta-minio-sqlserver/
│
├── 📄 docker-compose.yml              # SQL Server 2025 + MinIO
├── 📄 pyproject.toml                  # Dependências (UV)
├── 📄 .env.example                    # Template de variáveis
├── 📄 .python-version                 # Python 3.11
├── 📄 README.md                       # Este arquivo
├── 📄 mkdocs.yml                      # Configuração MkDocs
│
├── 📁 data/                           # Dados de exemplo (CSVs)
│   ├── regiao.csv
│   ├── estado.csv
│   ├── municipio.csv
│   ├── marca.csv
│   ├── modelo.csv
│   ├── cliente.csv
│   ├── endereco.csv
│   ├── telefone.csv
│   ├── carro.csv
│   ├── apolice.csv
│   └── sinistro.csv
│
├── 📁 notebook/                       # Jupyter Notebooks (Ordem: 00 → 03)
│   ├── 00_setup_sqlserver.ipynb       # Setup inicial (SQL Server)
│   ├── 01_sqlserver_to_minio_csv.ipynb # Extração → MinIO
│   ├── 02_csv_to_delta.ipynb          # Conversão → Delta Lake
│   └── 03_dml_delta.ipynb             # DML + History/Time Travel
│
└── 📁 docs/                           # Documentação MkDocs
    ├── index.md                       # Início
    ├── arquitetura.md                 # Arquitetura detalhada
    ├── setup.md                       # Instruções setup
    ├── notebooks.md                   # Guia de notebooks
    ├── ferramentas.md                 # Documentação de ferramentas
```

---

## 🔐 Credenciais Padrão

### SQL Server
- **Host**: `localhost`
- **Porta**: `1433`
- **Usuário**: `sa`
- **Senha**: `SqlServer@2025!`
- **Database**: `LojoDB`

### MinIO
- **Endpoint**: `http://localhost:9020`
- **Acesso**: `minioadmin`
- **Chave Secreta**: `minioadmin`
- **Console Web**: http://localhost:9021

### Variáveis de Ambiente (`.env`)
```bash
# SQL Server
DB_SERVER=localhost
DB_PORT=1433
DB_USER=sa
DB_PASSWORD=SqlServer@2025!
DB_DATABASE=LojoDB
DB_SCHEMA=dbo

# MinIO
MINIO_ENDPOINT=http://localhost:9020
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_LANDING_BUCKET=landing-zone
MINIO_BRONZE_BUCKET=bronze
```

---

## 🛠️ Tecnologias & Dependências

### Stack Principal

| Tecnologia | Versão | Descrição |
|-----------|--------|-----------|
| **Apache Spark** | 3.5.3 | Engine de processamento distribuído |
| **Delta Lake** | 3.2.0 | Formato lakehouse com ACID |
| **MinIO** | Latest | Object Storage (S3-compatible) |
| **SQL Server** | 2025 | Banco de dados relacional |
| **Python** | 3.11 | Linguagem de programação |
| **Java** | 11+ | Runtime Spark |

### Dependências Python

```toml
[project]
name = "spark-delta-minio-sqlserver"
version = "0.1.0"
dependencies = [
    "pyspark==3.5.3",
    "delta-spark==3.2.0",
    "boto3>=1.34.0",
    "pyodbc>=5.1.0",
    "pandas>=2.2.0",
    "jupyterlab>=4.2.5",
    "ipykernel>=6.29.0",
    "python-dotenv>=1.0.0",
    "mkdocs>=1.5.0",
    "mkdocs-material>=9.4.0",
]
```

---

## 📚 Conceitos Implementados

### 1. Arquitetura Medalhão

```
Landing Zone → Bronze → Silver → Gold
                ↓
         Dados Brutos      Dados Limpos      Dados Agregados
```

Este projeto implementa:
- ✅ **Landing Zone**: CSVs brutos do SQL Server
- ✅ **Bronze**: Delta Tables com estrutura básica

### 2. Delta Lake

**Características Demonstradas**:
- ✅ **ACID Transactions**: Garantias de consistência
- ✅ **Time Travel**: Acessar dados de versões anteriores
- ✅ **Schema Enforcement**: Validação automática de tipos
- ✅ **DML Operations**: INSERT, UPDATE, DELETE nativos
- ✅ **History**: Auditoria completa de mudanças

### 3. Object Storage (MinIO)

- ✅ Armazenamento escalável de dados
- ✅ API S3-compatível (boto3)
- ✅ Buckets para separação lógica de dados

### 4. Processamento Distribuído (Spark)

- ✅ Transformações com DataFrames
- ✅ Operações paralelas
- ✅ Otimização automática de queries

### 5. Integração de Dados

- ✅ SQL Server → Spark (ODBC/pyodbc)
- ✅ Spark → MinIO (boto3/S3 API)
- ✅ MinIO → Delta Lake (Spark)

---

## 🐳 Gerenciamento de Containers

### Status dos Containers

```bash
docker compose ps
```

### Logs em Tempo Real

```bash
# SQL Server
docker compose logs -f sqlserver-2025

# MinIO
docker compose logs -f minio
```

### Parar Containers

```bash
docker compose down
```

### Limpar Completamente (dados apagados)

```bash
docker compose down -v
```

### Reiniciar

```bash
docker compose restart
```

---

## 🔍 Monitoramento & Acesso

### MinIO Console

Acesse [http://localhost:9021](http://localhost:9021) para gerenciar buckets e objetos.

**Login**: `minioadmin` / `minioadmin`

### SQL Server Management Studio

Conecte com:
- **Server**: `localhost,1433`
- **Auth**: SQL Server
- **Login**: `sa`
- **Password**: `SqlServer@2025!`

### Jupyter Lab

```bash
jupyter lab
```

Acesso em `http://localhost:8888`

---

## 🐛 Troubleshooting

### Erro: "SQL Server não está respondendo"

```bash
# Aguarde 60 segundos após docker compose up
# Verificar logs
docker compose logs sqlserver-2025 | tail -20

# Reconectar
docker compose restart sqlserver-2025
```

### Erro: "ODBC Driver 18 not found"

```bash
# Ubuntu/Debian
sudo apt install -y msodbcsql18 unixodbc-dev

# Verificar instalação
odbcinst -q -d
```

### Erro: "Connection refused: localhost:1433"

```bash
# Verificar se container está rodando
docker compose ps

# Se não estiver, iniciar
docker compose up -d sqlserver-2025

# Aguardar ~60 segundos para SQL Server ficar pronto
```

### Erro: "MinIO bucket não existe"

Os buckets são criados automaticamente no primeiro notebook. Se não estiverem:

```bash
# Criar manualmente
aws s3 mb s3://landing-zone --endpoint-url http://localhost:9020
aws s3 mb s3://bronze --endpoint-url http://localhost:9020
```

### Kernel Jupyter não aparece

```bash
# Ativar venv e reinstalar ipykernel
source .venv/bin/activate
python -m ipykernel install --user --name=spark-delta-env
```

---

## 📖 Documentação MkDocs

Para visualizar a documentação completa:

```bash
# Instalar mkdocs (já incluído em pyproject.toml)
uv sync

# Servir documentação localmente
mkdocs serve

# Acessar em http://localhost:8000
```

---

## 💡 Exemplo de Código

### Leitura de Dados do SQL Server

```python
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, IntegerType
import pyodbc

spark = SparkSession.builder \
    .appName("Spark-SQL-Server") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

# Ler dados via JDBC
df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:sqlserver://localhost:1433;database=LojoDB") \
    .option("dbtable", "cliente") \
    .option("user", "sa") \
    .option("password", "SqlServer@2025!") \
    .load()

df.show()
```

### Escrita em Delta Lake

```python
from delta.tables import DeltaTable

# Escrever em Delta
df.write \
    .format("delta") \
    .mode("overwrite") \
    .option("path", "/minio/bronze/cliente") \
    .save()

# Ou criar tabela gerenciada
df.write \
    .format("delta") \
    .mode("overwrite") \
    .saveAsTable("cliente")
```

### Operação UPDATE em Delta

```python
from pyspark.sql.functions import col

delta_table = DeltaTable.forPath(spark, "/minio/bronze/cliente")

delta_table.update(
    condition = col("status") == "PENDENTE",
    set = {"status": "ATIVO", "updated_at": "2025-05-05"}
)
```

### Time Travel

```python
# Ver versão anterior
df_v0 = spark.read \
    .format("delta") \
    .option("versionAsOf", 0) \
    .load("/minio/bronze/cliente")

# Ou por timestamp
df_historical = spark.read \
    .format("delta") \
    .option("timestampAsOf", "2025-05-05 10:30:00") \
    .load("/minio/bronze/cliente")
```
