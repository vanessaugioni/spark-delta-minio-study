# Apache Spark

## Spark no pipeline

Neste projeto, o Apache Spark é a engine de processamento responsável por ler os dados de origem no MinIO e transformar essas cargas em tabelas Delta Lake.

Spark é usado para:

- ler arquivos CSV do bucket `landing-zone` do MinIO
- aplicar esquemas e conversões de tipos
- escrever tabelas Delta no bucket `bronze`
- executar operações DML de forma transacional em Delta Lake

## Versão utilizada

- Apache Spark 3.5.1
- PySpark como interface Python

## Fluxo de dados com Spark

A etapa principal de Spark é implementada em `notebook/01_csv_to_delta.ipynb`.

1. Spark lê arquivos CSV via S3A do MinIO.
2. Os dados são convertidos para DataFrames com os tipos corretos.
3. O DataFrame é gravado como Delta em MinIO `bronze`.

## Exemplo de leitura de CSV do MinIO

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName('SparkDeltaMinIO') \
    .getOrCreate()

csv_path = 's3a://landing-zone/departamentos/*.csv'
df = spark.read.option('header', 'true').csv(csv_path)
```

## Exemplo de escrita em Delta

```python
output_path = 's3a://bronze/departamentos'
df.write.format('delta').mode('overwrite').save(output_path)
```

## Integração com MinIO

Para que Spark acesse o MinIO, o notebook configura as propriedades S3A como:

- `fs.s3a.endpoint` para o endpoint do MinIO
- `fs.s3a.access.key` / `fs.s3a.secret.key`
- `fs.s3a.path.style.access` = `true`
- `fs.s3a.impl` para o conector S3A

## Por que usar Spark?

- Processa grandes volumes de dados de forma distribuída
- Permite conversão em Delta Lake com transações ACID
- Suporta leitura direta de objetos S3/MinIO
- Facilita operações analíticas e transformação ETL

## Notebooks relacionados

- `notebook/01_csv_to_delta.ipynb`: converte CSVs em Delta Lake
- `notebook/02_dml_delta.ipynb`: demonstra operações INSERT/UPDATE/DELETE e histórico Delta
