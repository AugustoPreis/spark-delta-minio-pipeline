# 🗄️ SQL Server - Relational Database

Guia prático para SQL Server neste projeto.

## O que é SQL Server?

SQL Server é um SGBD (Sistema Gerenciador de Banco de Dados) relacional da Microsoft.

```
SQL Server:
- Relacional (tabelas, chaves, relacionamentos)
- ACID nativo
- Transacional
- Comercial (ou Community Edition)
- Para produção
```

## Versão Utilizada

```
SQL Server 2025 (Developer Edition)
- Última versão
- Gratuita para desenvolvimento
- Feature completa
- Rodando em Docker
```

## Conectando

### Via pyodbc (Python)

```python
import pyodbc

# Conexão
connection_string = (
    'Driver={ODBC Driver 18 for SQL Server};'
    'Server=localhost;'
    'Database=LojoDB;'
    'UID=sa;'
    'PWD=SqlServer@2025!;'
    'Encrypt=no;'
    'TrustServerCertificate=yes;'
)

conn = pyodbc.connect(connection_string)
cursor = conn.cursor()

# Query
cursor.execute("SELECT * FROM cliente")
for row in cursor.fetchall():
    print(row)

# Inserir
cursor.execute("INSERT INTO cliente (nome) VALUES (?)", ("João",))
conn.commit()

# Fechar
cursor.close()
conn.close()
```

### Via Spark (JDBC)

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("SQLServer") \
    .getOrCreate()

# Ler tabela
df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:sqlserver://localhost:1433;database=LojoDB") \
    .option("dbtable", "cliente") \
    .option("user", "sa") \
    .option("password", "SqlServer@2025!") \
    .load()

# Query custom
df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:sqlserver://localhost:1433;database=LojoDB") \
    .option("query", "SELECT * FROM cliente WHERE status = 'ATIVO'") \
    .option("user", "sa") \
    .option("password", "SqlServer@2025!") \
    .load()

# Escrever tabela
df.write \
    .format("jdbc") \
    .option("url", "jdbc:sqlserver://localhost:1433;database=LojoDB") \
    .option("dbtable", "cliente_backup") \
    .option("user", "sa") \
    .option("password", "SqlServer@2025!") \
    .mode("overwrite") \
    .save()
```

### Via SQL Server Management Studio (SSMS)

```
1. Instalar SSMS (SQL Server Management Studio)
   - Download: https://learn.microsoft.com/sql/ssms/

2. Conectar:
   - Server: localhost,1433
   - Authentication: SQL Server Authentication
   - Login: sa
   - Password: SqlServer@2025!

3. Ver databases:
   - Esquerda → Databases
   - LojoDB deve aparecer após notebook 00
```

## Database Utilizado

### Database: LojoDB

```
LojoDB
├── Tables (11):
│   ├── regiao
│   ├── estado
│   ├── municipio
│   ├── marca
│   ├── modelo
│   ├── cliente
│   ├── endereco
│   ├── telefone
│   ├── carro
│   ├── apolice
│   └── sinistro
├── Indexes
├── Primary Keys
└── Foreign Keys
```

## Tabelas e Schemas

### Referência: regiao

```sql
CREATE TABLE regiao (
    id_regiao INT PRIMARY KEY,
    nome_regiao VARCHAR(50) NOT NULL
);

-- Exemplo de dados:
-- id_regiao | nome_regiao
-- 1         | Norte
-- 2         | Nordeste
-- 3         | Centro-Oeste
-- 4         | Sudeste
-- 5         | Sul
```

### Referência: estado

```sql
CREATE TABLE estado (
    id_estado INT PRIMARY KEY,
    nome_estado VARCHAR(50) NOT NULL,
    sigla VARCHAR(2) UNIQUE,
    id_regiao INT FOREIGN KEY REFERENCES regiao(id_regiao)
);

-- Exemplo de dados:
-- id_estado | nome_estado | sigla | id_regiao
-- 1         | São Paulo   | SP    | 4
-- 2         | Rio Janeiro | RJ    | 4
-- 3         | Minas Gerais| MG    | 4
```

### Transacional: cliente

```sql
CREATE TABLE cliente (
    id_cliente INT PRIMARY KEY,
    nome_cliente VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    data_cadastro DATETIME DEFAULT GETDATE(),
    status VARCHAR(20) DEFAULT 'ATIVO'
);

-- Exemplo de dados:
-- id_cliente | nome_cliente  | email           | data_cadastro | status
-- 1          | João Silva    | joao@email.com  | 2025-01-01    | ATIVO
-- 2          | Maria Santos  | maria@email.com | 2025-01-02    | ATIVO
```

## Operações SQL

### SELECT (Leitura)

```sql
-- Simples
SELECT * FROM cliente;

-- Com filtro
SELECT * FROM cliente WHERE status = 'ATIVO';

-- Contagem
SELECT COUNT(*) FROM cliente;

-- Agregação
SELECT status, COUNT(*) FROM cliente GROUP BY status;

-- Join
SELECT c.nome_cliente, e.nome_estado
FROM cliente c
LEFT JOIN endereco e ON c.id_cliente = e.id_cliente;
```

### INSERT (Inserção)

```sql
-- Simples
INSERT INTO cliente (id_cliente, nome_cliente, email)
VALUES (1001, 'João Silva', 'joao@example.com');

-- Múltiplos registros
INSERT INTO cliente (id_cliente, nome_cliente, email)
VALUES 
    (1002, 'Maria Santos', 'maria@example.com'),
    (1003, 'Pedro Costa', 'pedro@example.com');
```

### UPDATE (Atualização)

```sql
-- Atualizar um registro
UPDATE cliente
SET status = 'INATIVO'
WHERE id_cliente = 1;

-- Atualizar múltiplos
UPDATE cliente
SET status = 'INATIVO'
WHERE data_cadastro < '2023-01-01';
```

### DELETE (Deleção)

```sql
-- Deletar um registro
DELETE FROM cliente
WHERE id_cliente = 1;

-- Deletar com condição
DELETE FROM cliente
WHERE status = 'CANCELADO';
```

## Transações

```sql
-- Iniciar transação
BEGIN TRANSACTION;

-- Operações
INSERT INTO cliente VALUES (1001, 'João', ...);
UPDATE cliente SET status = 'ATIVO' WHERE id = 1001;

-- Commit (confirmar)
COMMIT;

-- Ou rollback (desfazer)
ROLLBACK;
```

## Performance

### Indexes

```sql
-- Criar índice
CREATE INDEX idx_cliente_status ON cliente(status);

-- Melhor performance em:
-- SELECT * FROM cliente WHERE status = 'ATIVO';

-- Deletar índice
DROP INDEX idx_cliente_status ON cliente;
```

### Execução Plan

```sql
-- Ver plano de execução
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT * FROM cliente WHERE status = 'ATIVO';

-- Vê: CPU time, elapsed time, logical reads
```

## Backup & Restore

### Backup (via SQL)

```sql
-- Backup completo
BACKUP DATABASE LojoDB
TO DISK = '/var/opt/mssql/backup/LojoDB.bak';

-- Backup de tabela específica
BACKUP DATABASE LojoDB
FILEGROUP = 'cliente'
TO DISK = '/var/opt/mssql/backup/cliente.bak';
```

### Restore

```sql
-- Restaurar database
RESTORE DATABASE LojoDB
FROM DISK = '/var/opt/mssql/backup/LojoDB.bak';

-- Restaurar para ponto no tempo
RESTORE DATABASE LojoDB
FROM DISK = '/var/opt/mssql/backup/LojoDB.bak'
WITH STOPAT = '2025-05-05 10:00:00';
```

## Administração

### Criar Database

```sql
CREATE DATABASE MinhaDB
ON PRIMARY
(
    NAME = 'MinhaDB_data',
    FILENAME = '/var/opt/mssql/data/MinhaDB_data.mdf',
    SIZE = 10MB,
    MAXSIZE = 1GB,
    FILEGROWTH = 1MB
)
LOG ON
(
    NAME = 'MinhaDB_log',
    FILENAME = '/var/opt/mssql/data/MinhaDB_log.ldf',
    SIZE = 5MB,
    MAXSIZE = 500MB,
    FILEGROWTH = 1MB
);
```

### Criar User

```sql
-- Login no servidor
CREATE LOGIN novo_usuario WITH PASSWORD = 'Senha@Forte123!';

-- User no database
CREATE USER novo_usuario FOR LOGIN novo_usuario;

-- Dar permissões
GRANT SELECT, INSERT, UPDATE ON DATABASE::LojoDB TO novo_usuario;
```

## Monitoramento

### Ver Conexões

```sql
SELECT * FROM sys.dm_exec_sessions;

-- Ou
sp_who2;
```

### Ver Queries em Execução

```sql
SELECT 
    session_id,
    start_time,
    status,
    command
FROM sys.dm_exec_requests
WHERE session_id > 50;
```

### Matar Conexão

```sql
-- Forçar kill de conexão
KILL 65;
```

## Troubleshooting

### Erro: "Login failed for user 'sa'"

```bash
# Verificar se SQL Server está rodando
docker compose ps sqlserver-2025

# Se não, iniciar
docker compose up -d sqlserver-2025

# Aguardar ~60 segundos
sleep 60
```

### Erro: "Connection timeout"

```bash
# Verificar logs
docker compose logs sqlserver-2025 | tail -50

# Se houver erro de configuração, reiniciar
docker compose restart sqlserver-2025
```

### Erro: "Database in use"

```sql
-- Forçar close de conexões
ALTER DATABASE LojoDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;

-- Depois
ALTER DATABASE LojoDB SET MULTI_USER;
```

## Integração com Projeto

### Fluxo

```
Notebook 00:
    └─ Cria database LojoDB
    └─ Cria 11 tabelas
    └─ Insere dados de exemplo
                ↓
Notebook 01:
    └─ Lê todas as 11 tabelas
    └─ Escreve em MinIO (CSV)
```

### Credenciais

```
Host: localhost
Port: 1433
User: sa
Password: SqlServer@2025!
Database: LojoDB
```
