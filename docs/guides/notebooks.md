# 📔 Guia de Notebooks

Descrição detalhada de cada notebook e o que esperar.

## Visão Geral da Sequência

```
00_setup_sqlserver.ipynb
    ↓ (SQL Server pronto com 11 tabelas)
01_sqlserver_to_minio_csv.ipynb
    ↓ (11 CSVs em landing-zone)
02_csv_to_delta.ipynb
    ↓ (11 Delta tables em bronze)
03_dml_delta.ipynb
    ↓ (Operações DML completas)
```

## Notebook 00: Setup SQLServer

**Arquivo**: `notebook/00_setup_sqlserver.ipynb`

**Objetivo**: Criar database `LojoDB` e carregar dados de exemplo.

### O que faz

1. **Conecta ao SQL Server** usando `pyodbc`
2. **Cria database** `LojoDB` (se não existir)
3. **Cria 11 tabelas** com esquemas pré-definidos
4. **Carrega dados de exemplo** em todas as tabelas

### Tabelas Criadas

| # | Tabela | Registros | Descrição |
|---|--------|-----------|-----------|
| 1 | `regiao` | 5 | Regiões geográficas brasileiras |
| 2 | `estado` | 27 | Estados brasileiros |
| 3 | `municipio` | 100+ | Municípios |
| 4 | `marca` | 10 | Marcas de veículos |
| 5 | `modelo` | 20 | Modelos de veículos |
| 6 | `cliente` | 1.000 | Clientes cadastrados |
| 7 | `endereco` | 1.000 | Endereços dos clientes |
| 8 | `telefone` | 500 | Telefones |
| 9 | `carro` | 500 | Carros do inventário |
| 10 | `apolice` | 200 | Apólices de seguro |
| 11 | `sinistro` | 50 | Sinistros registrados |

### Estrutura dos Dados

**Tabelas de Referência**:
```
regiao
├── id_regiao (PK)
└── nome_regiao

estado
├── id_estado (PK)
├── nome_estado
├── sigla
└── id_regiao (FK)
```

**Tabelas de Negócio**:
```
cliente
├── id_cliente (PK)
├── nome_cliente
├── email
├── data_cadastro
└── status

carro
├── id_carro (PK)
├── id_cliente (FK)
├── id_modelo (FK)
├── ano_fabricacao
└── preco
```

### Tempo Esperado

**~5-10 minutos** (primeira execução mais lenta)

### Outputs

- ✅ Database `LojoDB` criado
- ✅ 11 tabelas criadas
- ✅ ~3.000+ registros inseridos
- ✅ Mensagens de sucesso em verde

### Possíveis Erros

**"Login failed for user 'sa'"**
```python
# Aumentar wait time
import time
time.sleep(90)  # Aguardar SQL Server ficar pronto
```

**"Database already exists"**
- Normal em execuções subsequentes. Limpe se necessário:
```sql
DROP DATABASE LojoDB;
```

---

## Notebook 01: Extração para MinIO

**Arquivo**: `notebook/01_sqlserver_to_minio_csv.ipynb`

**Objetivo**: Extrair dados do SQL Server e salvar como CSV no MinIO.

### O que faz

1. **Conecta ao SQL Server** via `pyodbc`
2. **Lê todas as 11 tabelas** em DataFrames Spark
3. **Valida os dados** (contagem, tipos)
4. **Escreve em CSV** no MinIO bucket `landing-zone`

### Processo Detalhado

```python
# 1. Conexão
connection_string = "..."
df = spark.read.jdbc(...).load()

# 2. Validação
print(f"Registros lidos: {df.count()}")
df.printSchema()

# 3. Escrita
df.write \
    .format("csv") \
    .mode("overwrite") \
    .option("header", "true") \
    .save("s3a://landing-zone/tabela/")
```

### Saída Esperada

Estrutura no MinIO:

```
landing-zone/
├── regiao/
│   ├── part-00000.csv
│   └── _SUCCESS
├── estado/
│   ├── part-00000.csv
│   └── _SUCCESS
├── municipio/
│   ├── part-00000.csv
│   └── _SUCCESS
├── marca/
├── modelo/
├── cliente/
├── endereco/
├── telefone/
├── carro/
├── apolice/
└── sinistro/
```

### Tempo Esperado

**~2-5 minutos**

### Verificação no MinIO

1. Acesse http://localhost:9021
2. Login: `minioadmin` / `minioadmin`
3. Abra bucket `landing-zone`
4. Deve ver 11 pastas com arquivos CSV

### Possíveis Erros

**"Bucket landing-zone not found"**
- MinIO criará automaticamente

**"Connection timeout"**
- SQL Server ainda inicializando, aguarde 30 segundos

---

## Notebook 02: Conversão para Delta

**Arquivo**: `notebook/02_csv_to_delta.ipynb`

**Objetivo**: Converter CSVs em Delta Lake com garantias ACID.

### O que faz

1. **Lê CSVs** do MinIO `landing-zone`
2. **Valida schemas** (tipos de dados)
3. **Escreve em Delta** no MinIO `bronze`
4. **Cria metadados Delta** (log, versioning)

### Processo Detalhado

```python
# 1. Leitura CSV
df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .csv("s3a://landing-zone/cliente/")

# 2. Validação
assert df.count() > 0, "Nenhum registro!"
print(df.schema)

# 3. Escrita Delta
df.write \
    .format("delta") \
    .mode("overwrite") \
    .save("s3a://bronze/cliente/")
```

### O que é Delta Lake?

Delta Lake adiciona:

- **ACID Transactions**: Confiabilidade
- **Schema Enforcement**: Validação de tipos
- **Time Travel**: Histórico de versões
- **Unified Batch + Streaming**: Um formato para tudo

### Saída Esperada

Estrutura no MinIO:

```
bronze/
├── regiao/
│   ├── _delta_log/
│   │   ├── 00000000000000000000.json
│   │   └── _last_checkpoint
│   ├── part-00000.parquet
│   └── ...
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

### Arquivos Delta

- `_delta_log/`: Histórico de transações (JSON)
- `*.parquet`: Dados no formato Parquet comprimido
- `_SUCCESS`: Marcador de sucesso

### Tempo Esperado

**~3-8 minutos**

### Verificação

```python
from delta.tables import DeltaTable

# Verificar tabela Delta
delta_table = DeltaTable.forPath(spark, "s3a://bronze/cliente/")
print(delta_table.detail().show())
```

---

## Notebook 03: Operações DML

**Arquivo**: `notebook/03_dml_delta.ipynb`

**Objetivo**: Demonstrar INSERT, UPDATE, DELETE e Time Travel.

### Operações Demonstradas

#### 1. INSERT

```python
from pyspark.sql.functions import lit
from datetime import datetime

# Criar 100 novos clientes
new_clients = spark.createDataFrame([
    (1000 + i, f"Cliente_{i}", f"email_{i}@example.com", 
     datetime.now(), "ATIVO")
    for i in range(100)
], ["id_cliente", "nome_cliente", "email", 
    "data_cadastro", "status"])

# Inserir
delta_table = DeltaTable.forPath(spark, "s3a://bronze/cliente/")
delta_table.alias("t").merge(
    new_clients.alias("s"),
    "t.id_cliente = s.id_cliente"
).whenNotMatched().insert("*").execute()
```

#### 2. UPDATE

```python
# Atualizar 50 clientes (mudar status)
delta_table.update(
    condition = col("id_cliente") <= 50,
    set = {"status": "INATIVO"}
)
```

#### 3. DELETE

```python
# Deletar 10 clientes
delta_table.delete(
    condition = col("id_cliente") <= 10
)
```

#### 4. HISTORY

```python
# Ver histórico de mudanças
history = delta_table.history()
history.show()

# Mostra:
# version | timestamp | operation | user
# 0       | 10:00:00  | WRITE     | spark
# 1       | 10:01:00  | UPDATE    | spark
# 2       | 10:02:00  | DELETE    | spark
```

#### 5. TIME TRAVEL

```python
# Versão anterior (v0)
df_v0 = spark.read \
    .format("delta") \
    .option("versionAsOf", 0) \
    .load("s3a://bronze/cliente/")

# Por timestamp
df_past = spark.read \
    .format("delta") \
    .option("timestampAsOf", "2025-05-05 10:00:00") \
    .load("s3a://bronze/cliente/")
```

### Resultado das Operações

```
Initial state:    1.000 clientes
After INSERT:     1.100 clientes
After UPDATE:        50 marcados INATIVO
After DELETE:        10 deletados
Final state:      1.090 clientes
```

### Tempo Esperado

**~5-10 minutos**

### Conceitos Avançados

**MERGE (Upsert)**
```python
delta_table.alias("t").merge(
    updates.alias("u"),
    "t.id = u.id"
).whenMatched().update("*") \
 .whenNotMatched().insert("*") \
 .execute()
```

**VACUUM (Limpeza)**
```python
from delta.tables import DeltaTable
delta_table.vacuum(retention_hours=168)  # Manter 7 dias
```

**OPTIMIZE (Compactação)**
```python
delta_table.optimize().executeCompaction()
```

---

## Ordem de Execução

### ✅ Recomendado

1. **00 → 01 → 02 → 03** (sequência)
2. Execute "Run all cells" em cada

### ⚠️ Dependências

- **01 depende de 00** (SQL Server precisa estar pronto)
- **02 depende de 01** (CSVs precisam estar no MinIO)
- **03 depende de 02** (Delta tables precisam existir)

### 🔄 Repetição

Após rodar todos, você pode:

- Reexecutar **00** para limpar banco
- Reexecutar **01** para reextrair
- Etc.

---

## Dicas Importantes

### Selecionar Kernel Correto

Antes de executar:
- Clique no kernel (canto superior direito)
- Selecione "Python 3.11 (.venv)" ou "spark-delta-env"
- Nunca use "Python 3" padrão do sistema

### Monitorar Execução

- Procure por **✅** e mensagens verdes = sucesso
- Procure por **❌** e mensagens vermelhas = erro
- Leia stack traces completamente

### Tempos Longos

Primeira execução é lenta (JVM iniciando). Subsequentes são rápidas.

### Limpar Cache

Se parecer preso:
```python
spark.catalog.clearCache()
```
