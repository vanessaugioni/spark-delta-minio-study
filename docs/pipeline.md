# Pipeline de Dados

## Visão Geral do Pipeline

Este projeto implementa um pipeline de dados simplificado para demonstrar como integrar **Apache Spark**, **Delta Lake** e **MinIO** em uma arquitetura de dados moderna.

O fluxo principal é:

1. Dados de origem em CSV são copiados para o bucket `landing-zone` do MinIO.
2. Spark lê esses arquivos CSV diretamente de MinIO.
3. Os dados são convertidos para o formato **Delta Lake** na camada `bronze`.
4. Operações DML (INSERT, UPDATE e DELETE) são realizadas no Delta e exibem os recursos de histórico e time travel.

## Etapas do Pipeline

### 1. Ingestão para `landing-zone`

A camada `landing-zone` recebe os arquivos CSV de origem. Neste projeto, os CSVs originais estão em `data/` e são carregados automaticamente pelo serviço `minio-setup` definido em `docker-compose.yml`.

Arquivos usados:
- `data/departamentos.csv`
- `data/funcionarios.csv`
- `data/projetos.csv`
- `data/alocacoes.csv`

### 2. Conversão para Delta Lake (`bronze`)

O notebook `notebook/01_csv_to_delta.ipynb` realiza a conversão:

- lê CSVs do bucket `landing-zone`
- ajusta tipos e esquemas
- grava tabelas Delta no bucket `bronze`

Esta etapa cria uma base de tabelas Delta transacionais, pronta para consultas analíticas e atualizações confiáveis.

### 3. Operações DML e histórico Delta

O notebook `notebook/02_dml_delta.ipynb` demonstra:

- INSERT de novos registros
- UPDATE em registros existentes
- DELETE lógico/físico, conforme o caso
- consulta de histórico de versões
- uso de `time travel` para inspeção de dados antigos

## Arquitetura de Dados

A arquitetura deste pipeline pode ser representada como:

```
CSV files (data/) ──▶ MinIO landing-zone ──▶ Spark ──▶ MinIO bronze (Delta)
```

### Componentes

- MinIO: armazenamento S3 compatível para `landing-zone` e `bronze`
- Spark: processamento distribuído e leitura de dados via S3A
- Delta Lake: formato de tabela transacional com ACID, histórico e time travel
- JupyterLab: ambiente interativo para executar os notebooks

## Como executar

1. Subir o ambiente:

```bash
docker compose up -d
```

2. Criar o ambiente Poetry e instalar dependências:

```bash
poetry config virtualenvs.in-project true
poetry install
```

3. Executar os notebooks no JupyterLab ou via `poetry run jupyter lab`.

4. Rodar o notebook `notebook/01_csv_to_delta.ipynb` para criar as tabelas Delta.
5. Rodar o notebook `notebook/02_dml_delta.ipynb` para ver as operações DML e o histórico.

## Notebooks do pipeline

| Notebook | Descrição |
|---|---|
| `notebook/01_csv_to_delta.ipynb` | Converte CSVs do MinIO `landing-zone` para tabelas Delta em `bronze` |
| `notebook/02_dml_delta.ipynb` | Demonstra operações INSERT/UPDATE/DELETE e histórico Delta |

## Benefícios do pipeline

- Separação de camadas de ingestão e armazenamento
- Dados em formato aberto e transacional com Delta Lake
- Uso de MinIO como armazenamento S3 compatível
- Ambiente replicável via Docker e Poetry
