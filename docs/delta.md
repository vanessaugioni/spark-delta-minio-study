# Delta Lake

## O que é

**Delta Lake** é uma camada de armazenamento open-source que adiciona confiabilidade ao Data Lake. Ele traz transações **ACID**, controle de versão, schema enforcement e auditoria para arquivos Parquet armazenados em sistemas de arquivos locais ou na nuvem (S3, MinIO, ADLS, GCS).

Neste projeto, o Delta Lake é usado para armazenar as tabelas na camada **bronze** do MinIO, habilitando operações DML confiáveis diretamente sobre o Object Storage.

---

## Principais Recursos

| Recurso | Descrição |
|---|---|
| **Transações ACID** | Garante consistência mesmo em falhas ou escritas concorrentes |
| **Time Travel** | Consulta versões anteriores dos dados pelo número de versão ou timestamp |
| **Schema Enforcement** | Impede a escrita de dados com schema incompatível |
| **Schema Evolution** | Permite adicionar ou modificar colunas de forma controlada |
| **Delta Log** | Arquivo de transações JSON que registra cada operação na tabela |

---

## Como funciona

Cada tabela Delta é um diretório com arquivos Parquet e uma pasta `_delta_log/`. O Delta Log registra cada operação (add, remove, update) em arquivos JSON sequenciais. Isso é o que permite o Time Travel e as garantias ACID.

```
bronze/funcionarios/
├── part-00000-abc123.parquet   ← dados atuais
├── part-00000-def456.parquet   ← dados atuais
└── _delta_log/
    ├── 00000000000000000000.json  ← versão 0 (escrita inicial do CSV)
    ├── 00000000000000000001.json  ← versão 1 (INSERT de novos funcionários)
    ├── 00000000000000000002.json  ← versão 2 (UPDATE salarial)
    └── 00000000000000000003.json  ← versão 3 (soft DELETE)
```

---

## Implementação no Projeto

### 1. Configurar a SparkSession para Delta Lake + MinIO

```python
from pyspark.sql import SparkSession
from delta import configure_spark_with_delta_pip

builder = (
    SparkSession.builder
    .appName("Trabalho2_Delta")
    .master("local[*]")
    .config("spark.sql.extensions",
            "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog",
            "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    # Conexão com MinIO via S3A
    .config("spark.hadoop.fs.s3a.endpoint",         "http://localhost:9000")
    .config("spark.hadoop.fs.s3a.access.key",        "minioadmin")
    .config("spark.hadoop.fs.s3a.secret.key",        "minioadmin")
    .config("spark.hadoop.fs.s3a.path.style.access", "true")
    .config("spark.hadoop.fs.s3a.impl",
            "org.apache.hadoop.fs.s3a.S3AFileSystem")
    .config("spark.jars.packages",
            "org.apache.hadoop:hadoop-aws:3.3.4,"
            "com.amazonaws:aws-java-sdk-bundle:1.12.262")
)

spark = configure_spark_with_delta_pip(builder).getOrCreate()
```

### 2. Gravar uma tabela no formato Delta Lake

```python
BRONZE = "s3a://bronze"

df.write.format("delta").mode("overwrite").save(f"{BRONZE}/funcionarios")
```

### 3. Ler uma tabela Delta

```python
spark.read.format("delta").load(f"{BRONZE}/funcionarios").show()
```

---

## Operações DML

### INSERT — Adicionar registros

```python
from pyspark.sql import functions as F

novos = spark.createDataFrame([
    (9,  "Iris",  "QA Engineer", 5800.0,  1, 1),
    (10, "Jonas", "ML Engineer",  10500.0, 2, 1),
], ["id", "nome", "cargo", "salario", "departamento_id", "ativo"])

novos = novos.withColumn("_ingestao_ts", F.current_timestamp()) \
             .withColumn("_fonte", F.lit("manual"))

novos.write.format("delta").mode("append").save(f"{BRONZE}/funcionarios")
```

Resultado — estado da tabela após INSERT:

```
+---+-------+-------------+---------+
| id|   nome|        cargo|  salario|
+---+-------+-------------+---------+
|  1|  Alice|   Dev Senior|   8500.0|
|  2|    Bob|   Dev Junior|   4200.0|
|  3|  Carol|Data Engineer|   7300.0|
...
|  9|   Iris|  QA Engineer|   5800.0|
| 10|  Jonas|  ML Engineer|  10500.0|
+---+-------+-------------+---------+
```

### UPDATE — Atualizar registros

```python
from delta.tables import DeltaTable

delta_func = DeltaTable.forPath(spark, f"{BRONZE}/funcionarios")

# Reajuste de 10% para o departamento de Dados (id=2)
delta_func.update(
    condition="departamento_id = 2",
    set={"salario": "ROUND(salario * 1.10, 2)"}
)

# Promoção individual
delta_func.update(
    condition="id = 1",
    set={"cargo": "'Tech Lead'", "salario": "10000.0"}
)
```

Resultado — salários do departamento de Dados após UPDATE:

```
+---+-----+-------------+---------+
| id| nome|        cargo|  salario|
+---+-----+-------------+---------+
|  1|Alice|   Tech Lead |  10000.0|  ← promovida
|  3|Carol|Data Engineer|   8030.0|  ← reajuste +10%
|  5|  Eve|     Analista|   6050.0|  ← reajuste +10%
|  7|Grace|Data Scientist|10120.0|  ← reajuste +10%
+---+-----+-------------+---------+
```

### DELETE — Remover registros

```python
delta_proj = DeltaTable.forPath(spark, f"{BRONZE}/projetos")

delta_proj.delete("status = 'pausado'")
```

Resultado — tabela de projetos após DELETE:

```
+---+----------------------+--------------+
| id|                  nome|        status|
+---+----------------------+--------------+
|  1|  Pipeline Delta Lake | em_andamento |
|  2|   App Mobile TrailBR | em_andamento |
|  3| Migração Kubernetes  |    concluido |
|  4|Dashboard Analytics   | em_andamento |
+---+----------------------+--------------+
```

*(Projeto "Refactor API Gateway" com status `pausado` foi removido)*

---

## Time Travel

O Delta Lake mantém o histórico completo de todas as operações. É possível consultar qualquer versão anterior da tabela.

### Consultar o histórico

```python
delta_func = DeltaTable.forPath(spark, f"{BRONZE}/funcionarios")

delta_func.history().select(
    "version", "timestamp", "operation", "operationParameters"
).show(truncate=False)
```

Saída esperada:

```
+-------+-------------------+-----------+
|version|          timestamp|  operation|
+-------+-------------------+-----------+
|      3|2025-05-14 10:32:41|     UPDATE|  ← promoção Alice
|      2|2025-05-14 10:32:38|     UPDATE|  ← reajuste depto. Dados
|      1|2025-05-14 10:32:35|      WRITE|  ← INSERT novos funcionários
|      0|2025-05-14 10:32:30|      WRITE|  ← escrita inicial (CSV)
+-------+-------------------+-----------+
```

### Consultar uma versão anterior

```python
# Versão 0 — estado original logo após a extração do banco
spark.read.format("delta") \
    .option("versionAsOf", 0) \
    .load(f"{BRONZE}/funcionarios") \
    .select("id", "nome", "cargo", "salario") \
    .orderBy("id").show()
```

```
+---+------+-------------+-------+
| id|  nome|        cargo|salario|
+---+------+-------------+-------+
|  1| Alice|   Dev Senior| 8500.0|
|  2|   Bob|   Dev Junior| 4200.0|
|  3| Carol|Data Engineer| 7300.0|
|  4| David|          DBA| 6100.0|
|  5|   Eve|     Analista| 5500.0|
|  6| Frank|       DevOps| 7800.0|
|  7| Grace|Data Scientist|9200.0|
|  8|Heitor|   Dev Pleno | 6300.0|
+---+------+-------------+-------+
```

---

## Referências

- [Documentação oficial Delta Lake](https://docs.delta.io/)
- [delta-spark no PyPI](https://pypi.org/project/delta-spark/)
- [Canal DataWay BR](https://www.youtube.com/@DataWayBR)
- [jlsilva01/spark-delta-minio-sqlserver](https://github.com/jlsilva01/spark-delta-minio-sqlserver)