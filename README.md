# Projeto Apache Spark + Delta Lake + MinIO + SQLite

> Pipeline de dados com *Apache Spark*, *Delta Lake* e *MinIO* | Arquitetura de Dados | SATC

[![Python](https://img.shields.io/badge/python-3.11-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![Poetry](https://img.shields.io/badge/gerenciador-poetry-60A5FA)](https://python-poetry.org/)
[![Apache Spark](https://img.shields.io/badge/spark-3.5.1-E25A1C?logo=apachespark&logoColor=white)](https://spark.apache.org/)
[![Delta Lake](https://img.shields.io/badge/delta_lake-3.2.0-003366?logoColor=white)](https://delta.io/)
[![MinIO](https://img.shields.io/badge/minio-object_storage-C72E49?logo=minio&logoColor=white)](https://min.io/)
[![MkDocs](https://img.shields.io/badge/docs-mkdocs-526CFE?logo=materialformkdocs&logoColor=white)](https://vanessaugioni.github.io/spark-delta-minio-study)



## Participantes

| Nome | GitHub |
|---|---|
| Vanessa Ugioni | [@vanessaugioni](https://github.com/vanessaugioni) |
| Gabriel Muller | [@GabrielNM12](https://github.com/GabrielNM12) |
| Bettina da Silva | [@berbett](https://github.com/berbett) |


## Documentação

As explicações sobre as tecnologias utilizadas estão disponíveis no MkDocs do projeto:

🔗 **https://vanessaugioni.github.io/spark-delta-minio-study**



## Estrutura do Projeto

```
spark-delta-minio-study/
├── docker-compose.yml                  # MinIO (landing-zone + bronze)
├── pyproject.toml                      # Dependências gerenciadas pelo Poetry
├── poetry.lock
├── .gitignore
├── README.md
├── mkdocs.yml
├── docs/                               # Fontes do MkDocs
│   └── index.md
├── scripts/
│   └── init_db.py                      # Cria e popula o banco SQLite
└── notebooks/
    ├── tmp/                            # Gerado automaticamente pelo Spark
    ├── 01_extracao_landing_zone.ipynb  # Extração: SQLite → MinIO (CSV)
    ├── 02_csv_to_delta.ipynb           # Conversão: CSV → Delta Lake
    └── 03_dml_delta.ipynb              # DML: INSERT, UPDATE, DELETE
```



## Pré-requisitos

Instale as ferramentas abaixo antes de prosseguir:

| Ferramenta | Versão | Link |
|---|---|---|
| Java (JDK) | **17** | [adoptium.net](https://adoptium.net/temurin/releases/?version=17) |
| Python | **3.11** | [python.org](https://www.python.org/downloads/) |
| Poetry | **latest** | [python-poetry.org](https://python-poetry.org/docs/#installation) |
| Docker | **latest** | [docker.com](https://www.docker.com/products/docker-desktop/) |

> ⚠️ O Apache Spark **não é compatível com Java 21+**. Use obrigatoriamente o **JDK 17**.

### Verificar instalações

```bash
java -version         # openjdk version "17.x.x"
python --version      # Python 3.11.x
poetry --version      # Poetry (version x.x.x)
docker --version      # Docker version 24.x.x
docker compose version # Docker Compose version v2.x.x
```

### Configurar JAVA_HOME

**Linux**: adicione ao `~/.bashrc` ou `~/.zshrc` e reinicie o terminal:

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
```

> Após configurar, feche e reabra o terminal. Verifique com `echo $JAVA_HOME`.



## Instalação

### 1. Clonar o repositório

```bash
git clone https://github.com/vanessaugioni/spark-delta-minio-study.git
cd spark-delta-minio-study
```

### 2. Subir o MinIO com Docker

```bash
docker compose up -d
```

Aguarde alguns segundos. O container `minio-setup` cria os buckets `landing-zone` e `bronze` automaticamente.

> Console MinIO: **http://localhost:9001** — usuário: `minioadmin` / senha: `minioadmin`

### 3. Criar o ambiente virtual dentro do projeto

```bash
poetry config virtualenvs.in-project true
```

### 4. Instalar as dependências

```bash
poetry install
```

> Isso cria a pasta `.venv/` com todas as dependências do `pyproject.toml`. Requer conexão com a internet — o Spark também baixará pacotes Maven na primeira execução dos notebooks.

### 5. Criar o banco de dados SQLite

```bash
poetry run python scripts/init_db.py
```

> Isso cria o arquivo `notebooks/tmp/empresa.db` com as 3 tabelas e dados de exemplo.



## Dependências

Versões definidas no `pyproject.toml`:

```toml
[tool.poetry.dependencies]
python      = "^3.11"
pyspark     = "3.5.1"
delta-spark = "3.2.0"
jupyterlab  = "^4.0"
pandas      = "^2.0"
```

> Os pacotes `hadoop-aws` e `aws-java-sdk-bundle` são baixados via Maven pelo Spark na primeira execução dos notebooks para habilitar a integração com o MinIO (S3A).



## Executar os Notebooks

### Via JupyterLab

```bash
poetry run jupyter lab
```

Acesse **http://localhost:8888** e abra os notebooks na ordem:

| Notebook | Descrição |
|---|---|
| `notebooks/01_extracao_landing_zone.ipynb` | Lê as tabelas do SQLite e grava CSV no MinIO (`landing-zone`) |
| `notebooks/02_csv_to_delta.ipynb` | Lê os CSVs do MinIO e converte para Delta Lake (`bronze`) |
| `notebooks/03_dml_delta.ipynb` | Executa INSERT, UPDATE e DELETE; exibe HISTORY e TIME TRAVEL |

### Via VS Code

1. Instale a extensão [Jupyter](https://marketplace.visualstudio.com/items?itemName=ms-toolsai.jupyter)
2. Abra a pasta `spark-delta-minio-study` no VS Code
3. Abra um dos notebooks em `notebooks/`
4. Clique em **Select Kernel** → **Python Environments** → selecione `.venv`
5. Execute as células na ordem com `Shift + Enter`

> Em ambos os casos, execute as células **uma a uma**. A célula de configuração da SparkSession pode levar alguns minutos na primeira execução (download dos pacotes Maven).



## 📝 Referências

- 💻 [jlsilva01/spark-delta-minio-sqlserver](https://github.com/jlsilva01/spark-delta-minio-sqlserver)
- 📘 [Documentação Delta Lake](https://docs.delta.io/)
- 📘 [Documentação MinIO](https://min.io/docs/minio/linux/index.html)
- 📘 [Documentação PySpark](https://spark.apache.org/docs/latest/api/python/)
- 📘 [Documentação Poetry](https://python-poetry.org/docs/)


&nbsp;

<div align="center">
  <sub>Desenvolvido para a disciplina de Arquitetura de Dados — SATC</sub>
</div>
