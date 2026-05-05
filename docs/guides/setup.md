# ⚙️ Setup Detalhado

Instruções completas de setup para cada sistema operacional.

## Arquitetura do Ambiente

```
Seu Computador
├── Docker Engine
│   ├── SQL Server 2025 Container
│   │   └── Database: LojoDB (11 tabelas)
│   └── MinIO Container
│       ├── landing-zone/ (CSVs)
│       └── bronze/ (Delta tables)
├── Python 3.11 (.venv)
│   ├── PySpark 3.5.3
│   ├── Delta Spark 3.2.0
│   ├── JupyterLab
│   └── Outros...
└── Jupyter Lab (port 8888)
```

## Setup por Sistema Operacional

### Linux (Ubuntu 24.04 / Debian 12)

#### 1. Instalar Docker

```bash
# Remover versões antigas
sudo apt remove docker docker-engine docker.io containerd runc

# Adicionar repositório oficial
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) stable"

# Instalar Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Permitir uso sem sudo
sudo usermod -aG docker $USER
newgrp docker

# Validar
docker --version
docker compose version
```

#### 2. Instalar Java

```bash
sudo apt install -y openjdk-11-jdk

# Validar
java -version
```

#### 3. Instalar Python 3.11

```bash
# Verificar versão padrão
python3 --version

# Se não for 3.11+, instalar
sudo apt install -y python3.11 python3.11-venv

# Definir como padrão (opcional)
sudo update-alternatives --install /usr/bin/python3 python3 \
  /usr/bin/python3.11 1
```

#### 4. Instalar UV

```bash
# Instalar UV
curl -LsSf https://astral.sh/uv/install.sh | sh

# Adicionar ao PATH
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Validar
uv --version
```

#### 5. Instalar ODBC Driver

```bash
# Preparar repositório
curl https://packages.microsoft.com/keys/microsoft.asc | \
  sudo tee /etc/apt/trusted.gpg.d/microsoft.asc

sudo curl https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/prod.list | \
  sudo tee /etc/apt/sources.list.d/mssql-release.list

# Instalar
sudo apt update
sudo ACCEPT_EULA=Y apt install -y msodbcsql18 unixodbc-dev

# Validar
odbcinst -q -d
```

### macOS (Monterey+)

#### 1. Instalar Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### 2. Instalar Docker Desktop

```bash
# Opção 1: Homebrew
brew install docker docker-compose

# Opção 2: Download manual
# Acesse: https://www.docker.com/products/docker-desktop
# Instale e inicie a aplicação
```

#### 3. Instalar Java

```bash
brew install openjdk@11

# Criar symlink
sudo ln -sfn /opt/homebrew/opt/openjdk@11/libexec/openjdk.jdk \
  /Library/Java/JavaVirtualMachines/openjdk-11.jdk

# Validar
java -version
```

#### 4. Instalar Python 3.11

```bash
brew install python@3.11

# Symlink
brew link python@3.11

# Validar
python3.11 --version
```

#### 5. Instalar UV

```bash
brew install uv

# Validar
uv --version
```

#### 6. Instalar ODBC Driver

```bash
brew tap microsoft/mssql-release \
  https://github.com/Microsoft/homebrew-mssql-release

brew install msodbcsql18 mssql-tools18

# Validar
odbcinst -q -d
```

### Windows 11 (WSL2)

#### 1. Ativar WSL2

```powershell
# No PowerShell (Admin)
wsl --install -d Ubuntu-24.04

# Reiniciar computador
# Abrir WSL: Ubuntu 24.04 (do menu iniciar)
```

#### 2. Setup Inicial WSL

```bash
# Dentro do WSL
sudo apt update && sudo apt upgrade -y

# Seguir passos de "Linux" acima
```

#### 3. Docker Desktop

- Instale Docker Desktop for Windows
- Abra Settings → General → Marque "Use WSL 2"
- Abra Settings → Resources → WSL integration → Marque "Ubuntu-24.04"

## Criar Projeto

### Clone e Prepare

```bash
# Clonar
git clone https://github.com/AugustoPreis/spark-delta-minio-sqlserver.git
cd spark-delta-minio-sqlserver

# Estrutura do projeto
tree -L 2
```

### Subir Containers

```bash
# Iniciar containers
docker compose up -d

# Monitorar SQL Server
docker compose logs -f sqlserver-2025

# Pressione Ctrl+C quando ver:
# "Recovery is complete. This is an informational message only."
```

### Configurar Python

```bash
# Criar venv com UV
uv venv

# Ativar
source .venv/bin/activate

# Instalar dependências
uv sync

# Verificar
python -c "import pyspark; print('✅ PySpark OK')"
python -c "import delta; print('✅ Delta OK')"
```

### Configurar Variáveis

```bash
# Copiar template
cp .env.example .env

# Editar se necessário (padrões são adequados)
nano .env
```

## Verificação de Conectividade

### Teste SQL Server

```bash
python << 'EOF'
import pyodbc

try:
    conn = pyodbc.connect(
        'Driver={ODBC Driver 18 for SQL Server};'
        'Server=localhost;'
        'Database=master;'
        'UID=sa;'
        'PWD=SqlServer@2025!;'
    )
    print("✅ SQL Server conectado!")
    cursor = conn.cursor()
    cursor.execute("SELECT @@VERSION")
    version = cursor.fetchone()[0]
    print(f"   Versão: {version[:50]}...")
    conn.close()
except Exception as e:
    print(f"❌ Erro: {e}")
EOF
```

### Teste MinIO

```bash
pip install awscli  # Se não estiver instalado

aws s3 ls \
  --endpoint-url http://localhost:9020 \
  --access-key minioadmin \
  --secret-key minioadmin
```

### Teste Spark

```bash
python << 'EOF'
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("TestApp") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .getOrCreate()

print("✅ Spark iniciado com sucesso!")
print(f"   Versão: {spark.version}")

# Criar DataFrame simples
df = spark.createDataFrame([(1, "test")], ["id", "value"])
df.show()

spark.stop()
EOF
```

## Portas Utilizadas

| Serviço | Porta | URL | Credenciais |
|---------|-------|-----|------------|
| SQL Server | 1433 | `localhost:1433` | `sa` / `SqlServer@2025!` |
| MinIO API | 9020 | `localhost:9020` | `minioadmin` / `minioadmin` |
| MinIO Console | 9021 | http://localhost:9021 | `minioadmin` / `minioadmin` |
| Jupyter Lab | 8888 | http://localhost:8888 | Token (veja logs) |

## Variáveis de Ambiente

Arquivo `.env`:

```bash
# SQL Server
DB_SERVER=localhost           # Host do SQL Server
DB_PORT=1433                  # Porta padrão
DB_USER=sa                    # Admin account
DB_PASSWORD=SqlServer@2025!   # Senha padrão
DB_DATABASE=LojoDB            # Database a usar
DB_SCHEMA=dbo                 # Schema padrão

# MinIO (S3 Compatible)
MINIO_ENDPOINT=http://localhost:9020
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_LANDING_BUCKET=landing-zone
MINIO_BRONZE_BUCKET=bronze

# Spark (opcional)
SPARK_DRIVER_MEMORY=4g
SPARK_EXECUTOR_MEMORY=4g
```

## Limpeza / Reset

### Remover Tudo e Começar do Zero

```bash
# Parar containers
docker compose down

# Remover volumes (apaga dados)
docker compose down -v

# Remover containers, images, etc
docker system prune -a

# Iniciar novamente
docker compose up -d
```
