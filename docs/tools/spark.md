# 🛠️ Apache Spark

Guia prático para Apache Spark neste projeto.

## O que é Spark?

Apache Spark é um motor de processamento de dados distribuído que permite processar grandes volumes de dados rapidamente.

```
Spark = Hadoop 2.0

Hadoop (2005):           Spark (2011):
- MapReduce lento        - In-memory rápido
- Disco pesado           - Lazy evaluation
- Hadoop Streaming       - RDDs + DataFrames
- Complexo               - Simples + Poderoso
```

## Arquitetura Spark

### Master-Worker

```
┌──────────────────────┐
│  Spark Driver        │
│  (Master)            │
│  - Coordena jobs     │
│  - Gerencia estado   │
│  - Otimiza queries   │
└──────────────────────┘
        ↓↓↓
┌─────────────────────────────────────────┐
│  Cluster Manager                        │
│  (Local, YARN, Kubernetes, Mesos)       │
└─────────────────────────────────────────┘
        ↓↓↓
┌─────────────────────────────────────────┐
│  Executors (Workers)                    │
│  ├─ Executor 1  ├─ Executor 2           │
│  ├─ Executor 3  ├─ Executor 4           │
│  └─ ...         └─ ...                  │
└─────────────────────────────────────────┘
```

## Modo de Execução

### Local Mode (Este Projeto)

```python
spark = SparkSession.builder \
    .master("local[*]") \  # ← Tudo no seu PC
    .appName("MyApp") \
    .getOrCreate()

# Características:
# ✅ Simples (não precisa cluster)
# ✅ Debugging fácil
# ❌ Performance limitada
# ❌ Não escalável
```

### Cluster Mode (Produção)

```python
spark = SparkSession.builder \
    .master("spark://master:7077") \  # ← Apontando cluster
    .appName("MyApp") \
    .config("spark.executor.memory", "8g") \
    .config("spark.executor.cores", "4") \
    .config("spark.num.executors", "10") \
    .getOrCreate()

# Características:
# ✅ Escalável (múltiplas máquinas)
# ✅ Performance (paralelismo real)
# ❌ Mais complexo
# ❌ Requer infra
```

## Conceitos Chave

### 1. RDD (Resilient Distributed Dataset)

```python
# RDD é a abstração de baixo nível
rdd = spark.sparkContext.parallelize([1, 2, 3, 4, 5])

# Transformações (Lazy)
rdd2 = rdd.map(lambda x: x * 2)  # Não executa ainda
rdd3 = rdd2.filter(lambda x: x > 5)

# Action (Executa agora!)
result = rdd3.collect()  # [6, 8, 10]
```

### 2. DataFrame (API SQL)

```python
# DataFrame é como tabela SQL
df = spark.createDataFrame([
    (1, "João", 25),
    (2, "Maria", 30),
], ["id", "nome", "idade"])

# Transformações
df2 = df.filter(col("idade") > 25) \
        .select("nome", "idade")

# Action
df2.show()  # Exibe resultado
```

### 3. Lazy Evaluation

```python
# Nenhuma dessas operações executa:
df2 = df.filter(col("status") == "ATIVO")
df3 = df2.select("id", "nome")
df4 = df3.groupBy("nome").count()

# Só executa quando chama uma action:
df4.show()          # Action 1: Mostra resultado
count = df4.count() # Action 2: Conta linhas
df4.write...        # Action 3: Salva dados
```

## DataFrames em Detalhes

### Criação

```python
# Opção 1: De dados em memória
df = spark.createDataFrame([
    (1, "A", 100.0),
    (2, "B", 200.0),
], ["id", "letra", "valor"])

# Opção 2: De arquivo
df = spark.read.csv("arquivo.csv", header=True, inferSchema=True)
df = spark.read.json("arquivo.json")
df = spark.read.parquet("arquivo.parquet")

# Opção 3: De banco SQL
df = spark.read.jdbc(
    url="jdbc:sqlserver://...",
    table="tabela",
    properties={"user": "sa", "password": "..."}
)

# Opção 4: De Delta
from delta.tables import DeltaTable
df = spark.read.format("delta").load("s3://bronze/tabela/")
```

### Exploração

```python
# Schema (tipos de dados)
df.printSchema()
# root
#  |-- id: integer (nullable = true)
#  |-- nome: string (nullable = true)
#  |-- valor: double (nullable = true)

# Primeiras linhas
df.show(5)

# Estatísticas
df.describe().show()

# Info
df.info()  # Ou df.explain()
```

### Transformações

```python
# SELECT
df.select("id", "nome")

# WHERE
df.filter(col("valor") > 100)

# ORDER BY
df.orderBy(col("nome").desc())

# GROUP BY
df.groupBy("categoria").agg(
    count("*").alias("qtd"),
    sum("valor").alias("total")
)

# JOIN
df1.join(df2, "id", "inner")

# UNION
df1.union(df2)

# Adicionar coluna
df.withColumn("novo_valor", col("valor") * 1.1)

# Renomear coluna
df.withColumnRenamed("nome", "nome_cliente")

# Remover coluna
df.drop("nome")
```

### Actions

```python
# Contar linhas
df.count()

# Mostrar dados
df.show(10)

# Coletar em Python
data = df.collect()

# Salvar em arquivo
df.write.csv("output/")
df.write.parquet("output/")
df.write.format("delta").save("output/")

# Salvar em banco
df.write.jdbc(url=..., table="...", mode="overwrite")
```

## Otimizações

### Catalyst Optimizer

```python
# Spark otimiza automaticamente
df.filter(col("categoria") == "A") \
    .select("id", "nome") \
    .filter(col("valor") > 100)

# Spark reordena para:
df.filter(col("categoria") == "A") \
    .filter(col("valor") > 100) \
    .select("id", "nome")

# Motivo: filtros primeiro reduzem dados!
```

### Broadcast Join

```python
# Para juntar tabela grande com pequena
from pyspark.sql.functions import broadcast

large_df.join(
    broadcast(small_df),  # ← Pequena vai para todos workers
    "id",
    "inner"
)
```

### Particionamento

```python
# Ao salvar, particionar dados
df.write \
    .partitionBy("ano", "mes") \
    .mode("overwrite") \
    .parquet("s3://dados/")

# Estrutura:
# s3://dados/
# ├── ano=2025/mes=01/
# ├── ano=2025/mes=02/
# └── ano=2025/mes=03/

# Queries beneficiadas:
df = spark.read.parquet("s3://dados/")
df.filter((col("ano") == 2025) & (col("mes") == 1)).show()
# ✅ Só lê partição janeiro/2025
```

## Performance

### Monitoramento

```python
# Spark UI: http://localhost:4040
# Vê:
# - Jobs e stages
# - Tempo de execução
# - Shuffle memory
# - Gargalos

# Via código
df.explain(extended=True)  # Mostra plano de execução
```

### Cache

```python
# Manter DataFrame na memória
df.cache()

df.count()      # 1ª execução: Lê disco
df.count()      # 2ª execução: Usa cache (muito rápido)
df.count()      # 3ª execução: Usa cache

# Remover cache
df.unpersist()
```

## Configurações Importantes

### Session Config

```python
spark = SparkSession.builder \
    .appName("MyApp") \
    .config("spark.sql.shuffle.partitions", "200") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.skewJoin.enabled", "true") \
    .config("spark.sql.legacy.parquet.nanosAsLong", "true") \
    .getOrCreate()
```

### Variáveis de Ambiente

```bash
export SPARK_HOME=/opt/spark
export SPARK_DRIVER_MEMORY=4g
export SPARK_EXECUTOR_MEMORY=4g
export SPARK_EXECUTOR_CORES=4
export SPARK_NUM_EXECUTORS=2
```

## Debugging

### Modo Verbose

```python
spark.sparkContext.setLogLevel("DEBUG")

# Levels: OFF, FATAL, ERROR, WARN, INFO, DEBUG, TRACE
```

### Capturar Erros

```python
try:
    result = df.collect()
except Exception as e:
    print(f"Erro: {e}")
    print(f"Tipo: {type(e)}")
    import traceback
    traceback.print_exc()
```

## Integração com Delta

```python
# Ler Delta
df = spark.read.format("delta").load("s3://bronze/tabela/")

# Escrever Delta
df.write \
    .format("delta") \
    .mode("overwrite") \
    .save("s3://bronze/tabela/")

# DML em Delta
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "s3://bronze/tabela/")
delta_table.update(
    condition = col("id") == 1,
    set = {"status": "ATIVO"}
)
```

## SQL em Spark

```python
# Criar view
df.createOrReplaceTempView("cliente")

# Query SQL
result = spark.sql("""
    SELECT id, nome, COUNT(*) as qtd
    FROM cliente
    WHERE status = 'ATIVO'
    GROUP BY id, nome
""")

result.show()
```
