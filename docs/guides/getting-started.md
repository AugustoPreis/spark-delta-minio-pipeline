# 🚀 Começando com Spark + Delta Lake

Um guia rápido para colocar o projeto rodando em 15 minutos.

## Verificação de Pré-requisitos

Antes de começar, verifique se você tem instalado:

### Linux/WSL

```bash
# Verificar versões
docker --version              # Docker 20.10+
docker compose version        # Docker Compose v2+
python --version              # Python 3.11
java -version                # OpenJDK 11
git --version                # Git

# Verificar se pode usar docker sem sudo
docker ps
```

### macOS

```bash
# Com Homebrew
brew install docker docker-compose python@3.11 java

# Verificar
docker --version
docker compose version
python3 --version
java -version
```

## Instalação Passo a Passo

### Passo 1: Clonar o Repositório

```bash
git clone https://github.com/AugustoPreis/spark-delta-minio-sqlserver.git
cd spark-delta-minio-sqlserver

# Verificar estrutura
ls -la
```

### Passo 2: Subir Containers

```bash
# Iniciar SQL Server + MinIO
docker compose up -d

# Monitorar inicialização
docker compose logs -f sqlserver-2025

# Quando ver "[SQL Server 2025 is ready to accept connections]", pressione Ctrl+C
```

**Verificar status:**

```bash
docker compose ps

# Deve mostrar:
# NAME              STATUS
# sqlserver-2025    Up X seconds
# minio             Up X seconds
```

### Passo 3: Configurar Variáveis de Ambiente

```bash
# Copiar template
cp .env.example .env

# Editar se necessário (padrões já adequados)
cat .env
```

### Passo 4: Instalar Driver ODBC

#### Ubuntu/Debian

```bash
# Adicionar repositório
curl https://packages.microsoft.com/keys/microsoft.asc | \
  sudo tee /etc/apt/trusted.gpg.d/microsoft.asc

sudo curl https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/prod.list | \
  sudo tee /etc/apt/sources.list.d/mssql-release.list

# Instalar
sudo apt update
sudo ACCEPT_EULA=Y apt install -y msodbcsql18 unixodbc-dev

# Validar
odbcinst -q -d
# Deve mostrar: [ODBC Driver 18 for SQL Server]
```

#### macOS

```bash
brew tap microsoft/mssql-release https://github.com/Microsoft/homebrew-mssql-release
brew install msodbcsql18 mssql-tools18
```

### Passo 5: Ambiente Python

```bash
# Criar ambiente virtual
uv venv

# Ativar
source .venv/bin/activate

# Instalar dependências
uv sync

# Verificar instalação
python -c "import pyspark; print(f'PySpark {pyspark.__version__}')"
python -c "import delta; print(f'Delta {delta.__version__}')"
```

### Passo 6: Iniciar Jupyter Lab

```bash
# Ativar venv (se não estiver)
source .venv/bin/activate

# Iniciar Jupyter
jupyter lab

# Browser abrirá em http://localhost:8888
```

## Executar Primeiro Notebook

1. Abra Jupyter Lab (http://localhost:8888)
2. Navegue para `notebook/00_setup_sqlserver.ipynb`
3. **IMPORTANTE**: Selecione kernel `.venv`:
   - Clique no kernel (canto superior direito)
   - Selecione "Python 3.11 (.venv)" ou similar
4. Clique "Run all cells" ▶️

**Resultado esperado após ~5 minutos:**
- Database `LojoDB` criado
- 11 tabelas com dados carregados
- Mensagens de sucesso em verde

## Verificar Dados no MinIO

Acesse o console MinIO:

1. Abra http://localhost:9021 no browser
2. Login:
   - **Usuário**: `minioadmin`
   - **Senha**: `minioadmin`
3. Navegue até o bucket `landing-zone`
4. Após rodar notebook 01, você verá 11 arquivos CSV

## Próximo Notebook

Após sucesso do notebook 00:

1. Abra `notebook/01_sqlserver_to_minio_csv.ipynb`
2. Selecione kernel `.venv`
3. Clique "Run all cells" ▶️

Este notebook extrai dados de SQL Server → MinIO em CSV.

## Verificação Rápida

Para verificar se tudo está funcionando:

```bash
# 1. Verificar containers
docker compose ps

# 2. Verificar Python
python -c "import pyspark, delta; print('✅ Spark + Delta OK')"

# 3. Verificar conectividade
python -c "
import pyodbc
conn = pyodbc.connect(
    'Driver={ODBC Driver 18 for SQL Server};'
    'Server=localhost;'
    'Database=master;'
    'UID=sa;'
    'PWD=SqlServer@2025!;'
)
print('✅ SQL Server OK')
conn.close()
"

# 4. Verificar MinIO
aws s3 ls --endpoint-url http://localhost:9020 --no-sign-request
# Ou instale primeiro: pip install awscli
```

## Problemas Comuns

### Erro: "SQL Server Connection refused"

```bash
# SQL Server demora para ficar pronto
docker compose logs sqlserver-2025 | tail -20

# Aguarde 60-90 segundos e tente novamente
sleep 60
```

### Erro: "ODBC Driver 18 not found"

```bash
# Reinstalar driver
sudo apt remove -y msodbcsql18
sudo ACCEPT_EULA=Y apt install -y msodbcsql18

# Validar
odbcinst -q -d
```

### Kernel Jupyter não aparece

```bash
# Reinstalar ipykernel
source .venv/bin/activate
python -m ipykernel install --user --name spark-delta-env --display-name "Python 3.11 (Spark+Delta)"
```

### "Connection poolexhausted" em Spark

Aumentar conexões no driver ODBC:

```bash
# Editar /etc/odbcinst.ini
sudo nano /etc/odbcinst.ini

# Adicionar após [ODBC Driver 18 for SQL Server]:
# PoolingEnabled = 1
# MinPoolSize = 5
# MaxPoolSize = 20
```
