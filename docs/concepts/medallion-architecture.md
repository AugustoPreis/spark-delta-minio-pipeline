# 🏛️ Arquitetura Medalhão

Um padrão de design moderno para data lakes.

## O que é?

A **Arquitetura Medalhão** (Medallion Architecture) é um padrão de design para organizar dados em data lakes em camadas progressivas de qualidade.

```
        Medalha de Ouro
              ↑
         GOLD LAYER
         (Analytics)
              ↑
        Medalha de Prata
              ↑
         SILVER LAYER
         (Transformed)
              ↑
        Medalha de Bronze
              ↑
         BRONZE LAYER
         (Refined)
              ↑
       LANDING ZONE
       (Raw Data)
```

## As Camadas

### 1. Landing Zone (Zona de Pouso)

**Armazena**: Dados brutos, exatamente como vieram da fonte

```
landing-zone/
├── regiao.csv
├── estado.csv
├── municipio.csv
├── cliente.csv
└── ...

Características:
- ✅ Raw, sem transformação
- ✅ Imutável (append-only)
- ✅ Com metadados de origem
- ❌ Pode ter inconsistências
- ❌ Sem qualidade garantida
```

**Uso**: Investigação de dados, auditoria de ingestão

**Exemplo**:
```python
# Ler e armazenar exatamente como é
df = spark.read \
    .option("header", "true") \
    .csv("source_file.csv")

df.write.format("delta") \
    .mode("append") \
    .save("s3://landing-zone/dados_brutos/")
```

### 2. Bronze Layer (Medalha de Bronze)

**Armazena**: Dados estruturados, com tipagem básica

```
bronze/
├── regiao/
│   ├── _delta_log/
│   └── parquet files
├── estado/
├── cliente/
└── ...

Características:
- ✅ Esquema definido
- ✅ Tipos de dados corretos
- ✅ ACID transactions (Delta)
- ✅ Histórico de mudanças
- ✅ DML operations (INSERT/UPDATE/DELETE)
- ⚠️ Pode ter nulos e duplicados
- ⚠️ Sem validações de negócio
```

**Uso**: Deduplicação, limpeza básica, validações de integridade

**Exemplo**:
```python
# Converter landing zone para bronze
landing_df = spark.read.csv("s3://landing-zone/cliente/")

bronze_df = landing_df \
    .dropDuplicates() \
    .na.fill({"email": "sem-email@example.com"})

bronze_df.write.format("delta") \
    .mode("overwrite") \
    .save("s3://bronze/cliente/")
```

**Este projeto implementa até aqui** ✅

### 3. Silver Layer (Medalha de Prata)

**Armazena**: Dados transformados e validados

```
silver/
├── cliente_validado/
├── pedido_processado/
├── estoque_atualizado/
└── ...

Características:
- ✅ Tudo de Bronze
- ✅ Sem duplicatas garantido
- ✅ Sem nulos (ou com padrão)
- ✅ Validações de negócio aplicadas
- ✅ Relacionamentos estabelecidos
- ✅ Dados prontos para análise
```

**Uso**: Operacional, queries analíticas, dashboards

**Exemplo**:
```python
# Transformar bronze para silver
bronze_cliente = spark.read.format("delta") \
    .load("s3://bronze/cliente/")

bronze_endereco = spark.read.format("delta") \
    .load("s3://bronze/endereco/")

# Juntar e validar
silver_cliente = bronze_cliente \
    .join(bronze_endereco, "id_cliente", "left") \
    .filter(col("data_cadastro").isNotNull()) \
    .select("id_cliente", "nome", "email", 
            "endereco", "data_cadastro")

silver_cliente.write.format("delta") \
    .mode("overwrite") \
    .save("s3://silver/cliente/")
```

### 4. Gold Layer (Medalha de Ouro)

**Armazena**: Dados agregados para BI/Dashboards

```
gold/
├── vendas_por_regiao/
├── lucro_por_produto/
├── cliente_score/
└── ...

Características:
- ✅ Tudo de Silver
- ✅ Agregações completas
- ✅ Pronto para BI tools (PowerBI, Tableau)
- ✅ Otimizado para performance
- ✅ Particionado para queries rápidas
```

**Uso**: Dashboards, relatórios executivos, ML

**Exemplo**:
```python
# Criar agregações de ouro
silver_vendas = spark.read.format("delta") \
    .load("s3://silver/vendas/")

gold_vendas = silver_vendas \
    .groupBy("ano_mes", "regiao") \
    .agg(
        count("*").alias("qtd_vendas"),
        sum("valor").alias("receita"),
        avg("valor").alias("ticket_medio")
    )

gold_vendas.write.format("delta") \
    .partitionBy("ano_mes") \
    .mode("overwrite") \
    .save("s3://gold/vendas_resumo/")
```

## Benefícios

### 1. Governança de Dados

```
Landing Zone
  ├─ Quem: Engenheiro de Dados (ELT)
  ├─ O quê: Raw dumps
  └─ Quando: Sempre que houver dados novos

Bronze Layer
  ├─ Quem: Engenheiro de Dados
  ├─ O quê: Estruturação e deduplicação
  └─ Quando: Diariamente

Silver Layer
  ├─ Quem: Analista de Dados / Engenheiro
  ├─ O quê: Transformações de negócio
  └─ Quando: Por agendamento

Gold Layer
  ├─ Quem: BI Analyst / Cientista de Dados
  ├─ O quê: Agregações e ML features
  └─ Quando: Para relatórios e modelos
```

### 2. Escalabilidade

```
Produção hoje:
- 10 GB de dados
- 50 queries/dia
- 5 usuários

Crescimento esperado:
- 1 TB em 1 ano
- 1.000 queries/dia
- 100 usuários

✅ Medalhão permite crescimento sem reescrever código!
```

### 3. Qualidade de Dados

```
Landing → Bronze:     Estrutura
Bronze → Silver:      Qualidade
Silver → Gold:        Agregações

Cada camada garante
inputs corretos para
a próxima camada.
```

## Mapeamento de Responsabilidades

```
┌─────────────────────────────────────────┐
│       LANDING ZONE                      │
│  Responsável: Data Engineer              │
│  Ação: Ingerir dados brutos              │
│  Garantia: Dados legíveis                │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│       BRONZE LAYER                      │
│  Responsável: Data Engineer              │
│  Ação: Estruturação + Deduplicação      │
│  Garantia: ACID + Schema + Histórico     │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│       SILVER LAYER                      │
│  Responsável: Data/Analytics Engineer   │
│  Ação: Transformações de negócio        │
│  Garantia: Dados validados              │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│       GOLD LAYER                        │
│  Responsável: BI Analyst / Data Sci     │
│  Ação: Agregações + Features            │
│  Garantia: KPIs corretos                │
└─────────────────────────────────────────┘
```

## Fluxo de Dados Temporal

```
Dia 1:
Landing Zone ← Raw dump (10 GB)
    ↓
Bronze Layer ← Estruturação (9 GB, 5% duplicatas removidas)
    ↓
Silver Layer ← Validação (8 GB, 100% válido)
    ↓
Gold Layer ← Agregações (100 MB, 10 tabelas resumi)

Dia 2:
Landing Zone ← Raw dump (10 GB)
    ↓ (MERGE estratégico)
Bronze Layer ← Deduplicação incremental
    ↓
Silver Layer ← Updates...

A cada dia, a arquitetura se auto-renova!
```

## Anti-Patterns (O que NÃO fazer)

### ❌ Tudo em uma camada

```python
# Ruim
raw_data.filter(...) \
    .groupBy(...) \
    .agg(...) \
    .write.to_dashboard()

# Bom
landing = ingest_raw()
bronze = structure(landing)
silver = validate(bronze)
gold = aggregate(silver)
dashboard.read(gold)
```

### ❌ Dados vivos em Landing

```python
# Ruim - Landing Zone com Updates
landing_df.update(condition, values)

# Bom - Landing é imutável
landing_df.write.mode("append")
```

### ❌ Saltar camadas

```python
# Ruim - Bronze direto para ouro
bronze_df.groupBy(...).agg(...) \
    .write.save("s3://gold/")

# Bom - Ter silver como intermediária
silver_df = validate(bronze_df)
gold_df = aggregate(silver_df)
```

## Ferramentas por Camada

```
Landing Zone
├─ Ingestion: Airflow, Fivetran, AWS Glue
├─ Storage: S3, HDFS, MinIO
└─ Format: CSV, JSON, Parquet

Bronze Layer
├─ Processing: Spark, Flink
├─ Storage: S3, Delta Lake
└─ Format: Delta, Parquet

Silver Layer
├─ Processing: Spark, SQL
├─ Storage: S3, Data Warehouse
└─ Format: Delta, Iceberg

Gold Layer
├─ Processing: Spark, SQL, dbt
├─ Storage: Data Warehouse, Snowflake
└─ Tools: Power BI, Tableau, Superset
```

## Este Projeto vs Produção

### Este Projeto (Educacional)

```
Landing Zone ← SQL Server via JDBC
    ↓ (1 execução)
Bronze Layer ← Delta Lake
    ↓ (1 execução)
Dados finais com DML operations
```

**Tempo**: 30-40 minutos total

### Produção Real

```
Landing Zone ← Múltiplas fontes
    ├─ SQL Server (diário)
    ├─ APIs externas (tempo real)
    ├─ Cloud buckets (incremental)
    └─ Message queues (streaming)

Bronze Layer ← Processamento paralelo
    ├─ Deduplicação distribuída
    ├─ Tipagem automática
    ├─ Auditoria completa
    └─ Versionamento

Silver Layer ← Orquestração complexa
    ├─ Joins com múltiplas fontes
    ├─ Validações de negócio
    ├─ Enriquecimento de dados
    └─ Tratamento de anomalias

Gold Layer ← Agregações otimizadas
    ├─ Dashboards em tempo real
    ├─ Modelos de ML
    ├─ Relatórios automáticos
    └─ Alertas inteligentes

Refresh: Contínuo (24/7)
```
