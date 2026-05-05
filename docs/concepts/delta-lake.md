# 💾 Delta Lake - Deep Dive

Entenda os superpoderes do Delta Lake.

## O que é Delta Lake?

Delta Lake é uma camada de armazenamento que traz as garantias de um banco de dados tradicional para data lakes.

```
Traditional Database         Delta Lake          Regular Data Lake
├── ACID ✅                  ├── ACID ✅          ├── ACID ❌
├── Schema ✅                ├── Schema ✅        ├── Schema ❌
├── Updates ✅               ├── Updates ✅       ├── Updates ❌
├── Versions ✅              ├── Versions ✅      ├── Versions ❌
├── Scalability ❌           ├── Scalability ✅   ├── Scalability ✅
└── Cost ❌                  └── Cost ✅          └── Cost ✅
```

## Características Principais

### 1. ACID Transactions

**ACID** = Atomicity, Consistency, Isolation, Durability

```python
# Operação complexa com múltiplos passos
# Se algo falhar, TUDO é revertido

delta_table.alias("t").merge(
    updates.alias("u"),
    "t.id = u.id"
).whenMatched().delete() \
 .whenNotMatched().insert() \
 .execute()

# Se servidor cair no meio:
# ✅ Tudo foi feito
# ❌ Nada foi feito
# Nunca fica em estado inconsistente
```

### 2. Schema Enforcement

Valida tipos de dados automaticamente:

```python
# Criar tabela Delta com schema definido
schema = StructType([
    StructField("id", IntegerType(), False),
    StructField("nome", StringType(), True),
    StructField("preco", DoubleType(), True),
])

df.write \
    .schema(schema) \
    .format("delta") \
    .save("s3://bronze/produto/")

# Tentar inserir dado incorreto:
bad_df = spark.createDataFrame([
    (1, "Produto", "INVALIDO")  # preco é string!
], schema)

delta_table.insert(bad_df)  # ❌ ERRO: tipo incorreto!
```

### 3. Time Travel

Acessar dados de qualquer ponto no tempo:

```python
# Versão atual
df_now = spark.read \
    .format("delta") \
    .load("s3://bronze/cliente/")

# Versão anterior (v0)
df_v0 = spark.read \
    .format("delta") \
    .option("versionAsOf", 0) \
    .load("s3://bronze/cliente/")

# 2 versões atrás
df_v2_ago = spark.read \
    .format("delta") \
    .option("versionAsOf", current_version - 2) \
    .load("s3://bronze/cliente/")

# Em horário específico
df_at_10am = spark.read \
    .format("delta") \
    .option("timestampAsOf", "2025-05-05 10:00:00") \
    .load("s3://bronze/cliente/")
```

**Use Cases**:
- 🔍 Auditar mudanças
- 🐛 Debugar problemas
- 📊 Análises históricas
- 🔄 Reprocessar dados

### 4. DML Operations

Operações de banco de dados diretas:

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col

delta_table = DeltaTable.forPath(spark, "s3://bronze/cliente/")

# INSERT
new_data = spark.createDataFrame([(1001, "João")], ["id", "nome"])
delta_table.insert(new_data)

# UPDATE
delta_table.update(
    condition = col("status") == "PENDENTE",
    set = {"status": "ATIVO"}
)

# DELETE
delta_table.delete(col("id") == 1001)

# MERGE (Upsert)
delta_table.alias("t").merge(
    updates.alias("u"),
    "t.id = u.id"
).whenMatched().update("*") \
 .whenNotMatched().insert("*") \
 .execute()
```

### 5. History & Auditing

Rastrear todas as mudanças:

```python
# Ver histórico completo
history = delta_table.history()
history.show()

# Output:
# version | timestamp           | operation | user
# 0       | 2025-05-05 10:00:00 | WRITE     | spark
# 1       | 2025-05-05 10:01:00 | UPDATE    | spark
# 2       | 2025-05-05 10:02:00 | DELETE    | spark

# Ver detalhes de uma operação
history.filter(col("version") == 1).show()

# Buscar por usuário
history.filter(col("user") == "admin").show()

# Ver operações recentes
history.orderBy("timestamp").show(10)
```

## Estrutura Física

### Diretório Delta

```
s3://bronze/cliente/
├── _delta_log/                    # Transaction log
│   ├── 00000000000000000000.json  # Versão 0
│   ├── 00000000000000000001.json  # Versão 1
│   ├── 00000000000000000002.json  # Versão 2
│   └── _last_checkpoint           # Checkpoint
├── part-00000.parquet             # Data (v0)
├── part-00001.parquet             # Data (v1)
└── part-00002.parquet             # Data (v2)
```

### Transaction Log (JSON)

```json
{
  "add": {
    "path": "part-00000.parquet",
    "size": 1048576,
    "modificationTime": 1683270000000,
    "dataChange": true,
    "stats": "{\"numRecords\":1000,\"minValues\":{...},\"maxValues\":{...}}"
  }
}
```

Cada linha é uma ação atômica!

## Comparação: Parquet vs Delta

### Parquet (Formato de Arquivo)

```
file.parquet
├── Row Group 1
├── Row Group 2
└── Footer (metadata)

Características:
✅ Compressão eficiente
✅ Formato colunar
✅ Partições
❌ Sem versionamento
❌ Sem ACID
❌ Updates difíceis
```

### Delta Lake (Formato de Tabela)

```
delta_table/
├── _delta_log/        # ← DIFERENÇA!
│   └── 00000...json   # ← Histórico
├── part-00000.parquet
├── part-00001.parquet
└── ...

Características:
✅ Tudo do Parquet
✅ Versionamento
✅ ACID transactions
✅ Updates/Deletes
✅ Schema evolution
```

## Delta em Produção

### Caso de Uso: ETL Pipeline

```python
# 1. Ingerir dados brutos
raw_df = spark.read.parquet("s3://raw/data/")
raw_df.write.format("delta").mode("overwrite") \
    .save("s3://landing-zone/orders/")

# 2. Limpar e transformar
landing = spark.read.format("delta") \
    .load("s3://landing-zone/orders/")

clean_df = landing \
    .dropna() \
    .filter(col("amount") > 0) \
    .withColumn("date", to_date("timestamp"))

clean_df.write.format("delta").mode("overwrite") \
    .save("s3://bronze/orders/")

# 3. Agregar para BI
bronze = spark.read.format("delta") \
    .load("s3://bronze/orders/")

metrics = bronze \
    .groupBy("date", "category") \
    .agg(
        count("*").alias("count"),
        sum("amount").alias("total")
    )

metrics.write.format("delta").mode("overwrite") \
    .save("s3://gold/order_metrics/")

# 4. Versionar tudo
for table in ["landing-zone", "bronze", "gold"]:
    history = spark.read.format("delta") \
        .load(f"s3://{table}/orders/") \
        .history()
    print(f"{table}: {history.count()} versions")
```

### Compactação (Optimization)

```python
# Consolidar muitos pequenos arquivos em menos arquivos grandes
delta_table.optimize().executeCompaction()

# Antes: 1000+ arquivos pequenos (lento)
# Depois: 10 arquivos grandes (rápido)

# Com Z-order para queries mais rápidas
delta_table.optimize().executeCompaction(
    predicate = col("date") > "2025-01-01"
)
```

### Limpeza (Vacuum)

```python
# Remover arquivos deletados (economizar espaço)
delta_table.vacuum(retention_hours=168)  # Manter 7 dias

# Cuidado: Impossibilita Time Travel acima de 7 dias
# Use com cautela em produção!
```

## Schema Evolution

Adicionar/remover colunas sem problemas:

```python
# Versão 1: Tabela original
df_v1 = spark.createDataFrame([
    (1, "João"),
    (2, "Maria")
], ["id", "nome"])

df_v1.write.format("delta").save("s3://cliente/")

# Versão 2: Adicionar coluna
df_v2 = spark.createDataFrame([
    (3, "Pedro", 25)
], ["id", "nome", "idade"])

# Merge com schema evolution
delta_table.alias("t").merge(
    df_v2.alias("u"),
    "t.id = u.id"
).whenNotMatched().insertAll().execute()

# ✅ Delta permitiu nova coluna automaticamente!
```

## Performance

### Indexing (Z-Order)

```python
# Otimizar para queries frequentes em coluna "customer_id"
delta_table.optimize().executeCompaction(
    zOrderBy=["customer_id"]
)

# Resultado:
# SELECT * FROM cliente WHERE customer_id = 123
# ✅ Muito mais rápido (coluna está ordenada)
```

### Partitioning

```python
# Particionar por mês para queries mais rápidas
df.write \
    .format("delta") \
    .partitionBy("ano_mes") \
    .mode("overwrite") \
    .save("s3://bronze/vendas/")

# Estrutura:
# s3://bronze/vendas/
# ├── ano_mes=202501/
# ├── ano_mes=202502/
# └── ano_mes=202503/

# Query beneficiada:
# SELECT * FROM vendas WHERE ano_mes = 202501
# ✅ Só lê partição 202501 (muito rápido)
```

## Comparação com Alternativas

| Feature | Delta | Parquet | CSV | Iceberg |
|---------|-------|---------|-----|---------|
| ACID | ✅ | ❌ | ❌ | ✅ |
| Versionamento | ✅ | ❌ | ❌ | ✅ |
| Schema Enforcement | ✅ | ✅ | ❌ | ✅ |
| Updates/Deletes | ✅ | ❌ | ❌ | ✅ |
| Time Travel | ✅ | ❌ | ❌ | ✅ |
| Compressão | ✅ | ✅ | ❌ | ✅ |
| Custo | 🟢 Baixo | 🟢 Baixo | 🟡 Médio | 🔴 Alto |
| Adoção | 🔴 Alto | 🟢 Máximo | 🟡 Médio | 🟡 Crescendo |
