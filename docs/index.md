# 🏠 Bem-vindo ao Spark Delta Lake + MinIO + SQL Server

Este é um projeto educacional completo que demonstra uma **pipeline de dados moderna** utilizando as melhores práticas da indústria.

## 🎯 O que você aprenderá

Este projeto ensina:

- ✅ **Apache Spark** - Motor de processamento distribuído
- ✅ **Delta Lake** - Formato de armazenamento lakehouse com ACID
- ✅ **MinIO** - Object Storage compatível com S3
- ✅ **SQL Server** - Integração com bancos de dados relacionais
- ✅ **Arquitetura Medalhão** - Padrão de design moderno para data lakes
- ✅ **Operações DML** - INSERT, UPDATE, DELETE em Delta tables
- ✅ **Time Travel** - Versionamento de dados

## 🚀 Quick Start

### 1️⃣ Pré-requisitos

```bash
# Instalar dependências de sistema
docker --version          # Docker 20.10+
docker compose version    # Docker Compose v2+
python --version          # Python 3.11+
java -version            # OpenJDK 11+
```

### 2️⃣ Clonar repositório

```bash
git clone https://github.com/AugustoPreis/spark-delta-minio-sqlserver.git
cd spark-delta-minio-sqlserver
```

### 3️⃣ Subir infraestrutura

```bash
docker compose up -d
sleep 60  # Aguardar SQL Server ficar pronto
```

### 4️⃣ Configurar Python

```bash
uv venv
source .venv/bin/activate
uv sync
```

### 5️⃣ Executar notebooks

```bash
jupyter lab
# Abra os notebooks em ordem: 00 → 01 → 02 → 03
```

## 📊 Arquitetura em Camadas

```
┌─────────────────────────────────────────────────────────┐
│         PIPELINE DE DADOS (Arquitetura Medalhão)       │
└─────────────────────────────────────────────────────────┘

Landing Zone          Bronze Layer          Silver/Gold
(Raw CSV)          (Delta ACID)          (Aggregated)
    ↓                  ↓                        ↓
[CSV Files]    [Delta Tables]    [Analytics/Reports]

Fonte: SQL Server
Destino: MinIO Buckets
Processamento: Apache Spark
```

## 📔 Estrutura dos Notebooks

| # | Nome | Descrição | Tempo |
|---|------|-----------|-------|
| 00 | Setup SQLServer | Cria database e carrega dados | 5-10 min |
| 01 | Extração para MinIO | Lê SQL Server, escreve CSV | 2-5 min |
| 02 | Conversão Delta | CSV → Delta Lake | 3-8 min |
| 03 | Operações DML | INSERT, UPDATE, DELETE | 5-10 min |

## 🎨 Principais Conceitos

### Delta Lake ✨

Delta Lake é uma camada de armazenamento que traz confiabilidade em data lakes:

- **ACID Transactions**: Garantias de consistência
- **Schema Enforcement**: Validação automática de tipos
- **Time Travel**: Viaje no tempo entre versões
- **DML Operations**: Operações de banco de dados em dados grandes

### Arquitetura Medalhão 🏛️

Um padrão de design moderno para data lakes:

1. **Landing Zone**: Dados brutos extraídos (sem transformação)
2. **Bronze**: Dados com limpeza básica e estrutura
3. **Silver**: Dados transformados e validados
4. **Gold**: Dados agregados e prontos para BI

Este projeto implementa Landing Zone → Bronze.

## 🔧 Credenciais Padrão

| Serviço | Usuário | Senha | Porta |
|---------|---------|-------|-------|
| SQL Server | `sa` | `SqlServer@2025!` | 1433 |
| MinIO | `minioadmin` | `minioadmin` | 9020 |
| MinIO Console | - | - | 9021 |

> ⚠️ **Nota**: Use credenciais fortes em produção!

## 📚 Próximos Passos

1. **[Setup Completo](guides/setup.md)** - Instruções passo a passo
2. **[Executar Notebooks](guides/notebooks.md)** - Guia de execução
3. **[Entender Delta Lake](concepts/delta-lake.md)** - Aprofundar conhecimento
4. **[Troubleshooting](tutorials/troubleshooting.md)** - Resolver problemas

## 💡 Exemplos Rápidos

### Conectar ao SQL Server com Spark

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MyApp") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .getOrCreate()

df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:sqlserver://localhost:1433;database=LojoDB") \
    .option("dbtable", "cliente") \
    .option("user", "sa") \
    .option("password", "SqlServer@2025!") \
    .load()
```

### Escrever em Delta Lake

```python
df.write \
    .format("delta") \
    .mode("overwrite") \
    .save("/minio/bronze/cliente")
```

### Time Travel

```python
# Ver versão anterior
df_v0 = spark.read \
    .format("delta") \
    .option("versionAsOf", 0) \
    .load("/minio/bronze/cliente")
```
