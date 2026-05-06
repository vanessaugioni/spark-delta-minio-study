# MinIO

## O que é

**MinIO** é um Object Storage de alta performance, compatível com a API do Amazon S3. Ele permite armazenar qualquer tipo de dado não estruturado — arquivos, logs, backups, imagens — usando os mesmos padrões de acesso do S3, porém rodando localmente ou em infraestrutura própria.

Neste projeto, o MinIO funciona como a camada de armazenamento do Data Lakehouse, substituindo o S3 da AWS em ambiente local para desenvolvimento e estudo.

---

## Conceitos Principais

| Conceito | Descrição |
|---|---|
| **Bucket** | Container lógico para armazenar objetos (equivalente a uma pasta raiz no S3) |
| **Objeto** | Qualquer arquivo armazenado no MinIO (CSV, Parquet, JSON, etc.) |
| **API S3** | Interface de acesso compatível com Amazon S3 |
| **Console** | Interface web para gerenciar buckets e objetos visualmente |
| **S3A** | Protocolo usado pelo Hadoop/Spark para acessar storage compatível com S3 |

---

## Buckets do Projeto

Este projeto utiliza dois buckets que representam as camadas da Arquitetura Medalhão:

### `landing-zone`

Camada de entrada dos dados brutos. Recebe os arquivos CSV extraídos diretamente do banco SQLite, sem nenhuma transformação.

```
landing-zone/
├── departamentos/
│   └── part-00000-abc.csv     ← dados brutos, tipos como string
├── funcionarios/
│   └── part-00000-def.csv
└── projetos/
    └── part-00000-ghi.csv
```

### `bronze`

Camada de dados convertidos para o formato Delta Lake. Os dados já têm tipos corretos e metadados de ingestão adicionados.

```
bronze/
├── departamentos/
│   ├── part-00000-abc.parquet
│   └── _delta_log/
│       └── 00000000000000000000.json
├── funcionarios/
│   ├── part-00000-def.parquet
│   └── _delta_log/
│       ├── 00000000000000000000.json  ← escrita inicial
│       ├── 00000000000000000001.json  ← INSERT
│       ├── 00000000000000000002.json  ← UPDATE
│       └── 00000000000000000003.json  ← DELETE
└── projetos/
    ├── part-00000-ghi.parquet
    └── _delta_log/
```

---

## Setup com Docker

O MinIO é iniciado via Docker Compose. O container `minio-setup` cria os buckets automaticamente ao subir o ambiente.

```yaml
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"   # API S3 (usada pelo Spark)
      - "9001:9001"   # Console Web

  minio-setup:
    image: minio/mc:latest
    entrypoint: >
      /bin/sh -c "
        sleep 5;
        mc alias set local http://minio:9000 minioadmin minioadmin;
        mc mb --ignore-existing local/landing-zone;
        mc mb --ignore-existing local/bronze;
      "
```

### Iniciar

```bash
docker compose up -d
```

### Parar

```bash
docker compose down
```

### Acessar o Console Web

Abra no navegador: **http://localhost:9001**

| Campo | Valor |
|---|---|
| Usuário | `minioadmin` |
| Senha | `minioadmin` |

---

## Integração com o Spark (S3A)

O Spark acessa o MinIO através do protocolo **S3A**, que é a implementação Hadoop para sistemas compatíveis com S3. As configurações necessárias na `SparkSession` são:

```python
.config("spark.hadoop.fs.s3a.endpoint",         "http://localhost:9000")
.config("spark.hadoop.fs.s3a.access.key",        "minioadmin")
.config("spark.hadoop.fs.s3a.secret.key",        "minioadmin")
.config("spark.hadoop.fs.s3a.path.style.access", "true")
.config("spark.hadoop.fs.s3a.impl",
        "org.apache.hadoop.fs.s3a.S3AFileSystem")
.config("spark.hadoop.fs.s3a.aws.credentials.provider",
        "org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider")
.config("spark.jars.packages",
        "org.apache.hadoop:hadoop-aws:3.3.4,"
        "com.amazonaws:aws-java-sdk-bundle:1.12.262")
```

!!! warning "Atenção: path.style.access"
    A configuração `path.style.access = true` é obrigatória para o MinIO. Sem ela, o Spark tentaria acessar o bucket como subdomínio (ex: `landing-zone.localhost:9000`), o que não funciona em ambiente local.

---

## Leitura e Escrita via Spark

### Gravar CSV no landing-zone

```python
df.coalesce(1) \
  .write \
  .mode("overwrite") \
  .option("header", "true") \
  .csv("s3a://landing-zone/funcionarios")
```

### Ler CSV do landing-zone

```python
df = spark.read \
    .option("header", "true") \
    .csv("s3a://landing-zone/funcionarios")
```

### Gravar Delta Lake no bronze

```python
df.write \
  .format("delta") \
  .mode("overwrite") \
  .save("s3a://bronze/funcionarios")
```

### Ler Delta Lake do bronze

```python
df = spark.read \
    .format("delta") \
    .load("s3a://bronze/funcionarios")
```

---

## MinIO vs. Amazon S3

| Característica | MinIO | Amazon S3 |
|---|---|---|
| Ambiente | Local / on-premise | Nuvem AWS |
| Custo | Gratuito | Pago por uso |
| API | Compatível com S3 | S3 nativo |
| Uso | Desenvolvimento / estudo | Produção |
| Configuração Spark | `fs.s3a.endpoint` + `path.style.access=true` | Apenas credenciais AWS |

!!! tip "Migração para produção"
    Como o MinIO é 100% compatível com a API S3, migrar o projeto para a AWS em produção exige apenas remover a configuração `fs.s3a.endpoint` e fornecer as credenciais AWS. O código de leitura e escrita permanece idêntico.

---

## Referências

- [Documentação oficial MinIO](https://min.io/docs/minio/linux/index.html)
- [MinIO com Spark/S3A](https://min.io/docs/minio/linux/integrations/using-spark-with-minio.html)
- [Hadoop S3A — Documentação](https://hadoop.apache.org/docs/stable/hadoop-aws/tools/hadoop-aws/index.html)
- [jlsilva01/spark-delta-minio-sqlserver](https://github.com/jlsilva01/spark-delta-minio-sqlserver)