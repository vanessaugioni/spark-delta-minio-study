# Projeto: Apache Spark com Minio e SQL

> Pipeline de dados com *Apache Spark*, *Delta Lake*, *MinIO* com *SQL*| Arquitetura de Dados | SATC

[![Python](https://img.shields.io/badge/python-3.11-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![Poetry](https://img.shields.io/badge/gerenciador-poetry-60A5FA)](https://python-poetry.org/)
[![Apache Spark](https://img.shields.io/badge/spark-3.5.1-E25A1C?logo=apachespark&logoColor=white)](https://spark.apache.org/)
[![Delta Lake](https://img.shields.io/badge/delta_lake-3.2.0-003366?logoColor=white)](https://delta.io/)
[![MinIO](https://img.shields.io/badge/minio-object_storage-C72E49?logo=minio&logoColor=white)](https://min.io/)
[![MkDocs](https://img.shields.io/badge/docs-mkdocs-526CFE?logo=materialformkdocs&logoColor=white)](https://vanessaugioni.github.io/datalakehouse-study)


## Participantes

| Nome | GitHub |
|---|---|
| Vanessa Ugioni | [@vanessaugioni](https://github.com/vanessaugioni) |
| Gabriel Muller | [@GabrielNM12](https://github.com/GabrielNM12) |
| Bettina da Silva | [@berbett](https://github.com/berbett) |


## Documentação 
As explicações sobre as tecnologias utilizadas estão disponíveis no MkDocs do projeto:

🔗 https://vanessaugioni.github.io/spark-delta-minio-study


## Estrutura do Projeto

```
datalakehouse-study/
├── docs/                     # Fontes do MkDocs
├── data/                # Base de dados
    ├── alocacoes.csv
    ├── adepartamentos.csv
    ├── funcionarios.csv
    ├── projetos.csv
├── notebooks/
    ├── tmp                   # Gerado automaticamente pelo Spark
    ├── 01_csv_to_delta.ipynb      # Notebook Delta Lake - CSV
    ├── 02_dml_delta.ipynb         # Notebook Delta Lake - DML
├── warehouse/                # Gerado automaticamente pelo Spark
├── poetry.lock
├── pyproject.toml            # Dependências gerenciadas pelo Poetry
└── README.md
```

## Sobre o Projeto

Este repositório está organizado para executar o pipeline em notebooks Jupyter, usando arquivos CSV em `data/` que são importados para o bucket `landing-zone` do MinIO. A conversão para Delta Lake é feita no bucket `bronze` e as operações DML são realizadas diretamente em notebooks.

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
java -version
python --version
poetry --version
docker --version
docker compose version
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

O serviço `minio-setup` cria os buckets `landing-zone` e `bronze`, e importa os arquivos CSV de `data/` para o bucket `landing-zone`.

> Console MinIO: **http://localhost:9001** — usuário: `minioadmin` / senha: `minioadmin`

### 3. Criar o ambiente virtual dentro do projeto

```bash
poetry config virtualenvs.in-project true
```

### 4. Instalar as dependências

```bash
poetry install
```

> Isso cria a pasta `.venv/` com todas as dependências do `pyproject.toml`.

### 5. Deploy da documentação MkDocs

Use o ambiente Poetry para garantir que o `mkdocs-material` e `pymdown-extensions` estejam ativos:

```bash
poetry run mkdocs gh-deploy
```

## Dependências

Versões definidas no `pyproject.toml`:

```toml
[tool.poetry.dependencies]
python = "^3.11"
pyspark = "3.5.1"
delta-spark = "3.2.0"
jupyterlab = "^4.0"
pandas = "^2.0"
```

## Executar os Notebooks

### Via JupyterLab

```bash
poetry run jupyter lab
```

Acesse **http://localhost:8888** e abra, em ordem:

| Notebook | Descrição |
|---|---|
| `notebook/01_csv_to_delta.ipynb` | Converte os CSVs do MinIO (`landing-zone`) para Delta Lake (`bronze`) |
| `notebook/02_dml_delta.ipynb` | Executa `INSERT`, `UPDATE` e `DELETE` em uma tabela Delta |

### Via VS Code

1. Instale a extensão [Jupyter](https://marketplace.visualstudio.com/items?itemName=ms-toolsai.jupyter)
2. Abra a pasta do projeto no VS Code
3. Abra um dos notebooks em `notebook/`
4. Selecione o kernel `.venv`
5. Execute as células na ordem com `Shift + Enter`

> Execute as células uma a uma. A inicialização da SparkSession pode demorar alguns minutos enquanto baixa dependências Maven.

## Observações

- Os arquivos CSV são carregados diretamente de `data/` para o bucket `landing-zone` no MinIO.
- O notebook `01_csv_to_delta.ipynb` realiza a conversão para Delta Lake.
- O notebook `02_dml_delta.ipynb` executa operações transacionais e mostra o histórico.

## 📝 Referências

- 💻 [jlsilva01/spark-delta-minio-sqlserver](https://github.com/jlsilva01/spark-delta-minio-sqlserver)
- 📘 [Documentação Delta Lake](https://docs.delta.io/)
- 📘 [Documentação MinIO](https://min.io/docs/minio/linux/index.html)
- 📘 [Documentação PySpark](https://spark.apache.org/docs/latest/api/python/)
- 📘 [Documentação Poetry](https://python-poetry.org/docs/)

> Desenvolvido para a disciplina de Arquitetura de Dados — SATC
