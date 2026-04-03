# Big Data Assignment 2 Report

## 1. Goal

The goal of this assignment was to build a small search engine for text documents in a Hadoop environment.

The project follows these rules:

- data preparation is done with PySpark
- the cluster has a master node and a worker node
- indexing is done with Hadoop MapReduce
- the final index is stored in Cassandra
- search results are ranked with BM25

I did not use pandas or any other single-machine data processing library for the data pipeline.

## 2. Cluster Setup

The system runs with Docker Compose.

The container setup is:

- `cluster-master`
- `cluster-slave-1`
- `cassandra-server`

The master container starts the whole workflow. It launches Hadoop services, prepares the data, builds the index, loads the index into Cassandra, and runs a sample search.

The worker container is used by Hadoop YARN and HDFS as the slave node.

The Cassandra container stores the index tables.

The repo also uses one shared Docker image for the master and the worker. This keeps the Python runtime the same on both nodes.

## 3. Corpus

The corpus contains 100 text documents.

The source data can come from:

- `app/a.parquet` in HDFS, if it exists
- the bundled fallback corpus in `app/data`

The fallback corpus is a folder with many plain text files. Each file name contains the document id and the title.

The project uses only 100 documents. This keeps the pipeline light enough for Docker Desktop and still satisfies the assignment size.

## 4. Data Preparation

Data preparation is implemented in [app/prepare_data.py](/Users/jeanne/code%20projects/BD_ass2_new/app/prepare_data.py).

This step uses PySpark to:

- read the source corpus
- select the `id`, `title`, and `text` fields
- remove empty rows
- sample 100 documents

The code path is:

1. Read `/a.parquet` from HDFS if it is present.
2. Otherwise reuse the bundled text corpus in `app/data`.
3. Keep only rows where `text` is not null and not empty.
4. Sample up to 100 documents.
5. Write local `.txt` files for the chosen sample.
6. Upload those files to HDFS under `/data`.
7. Read `/data/*.txt` back with Spark and build one HDFS text file at `/input/data`.

The final record format written to HDFS is:

```text
doc_id<TAB>title<TAB>text
```

The output in HDFS is a standard Hadoop text output directory:

- `/input/data/_SUCCESS`
- `/input/data/part-00000`

This file is the input for the first MapReduce pipeline.

## 5. MapReduce Pipeline 1

Pipeline 1 is started by [app/create_index.sh](/Users/jeanne/code%20projects/BD_ass2_new/app/create_index.sh).

The input is `/input/data`, where each line is:

```text
doc_id<TAB>title<TAB>text
```

The first mapper reads one document and emits term-level records. The key point is:

- the term is the key for posting data
- the mapper also keeps document metadata for later steps

The reducer groups the records by term and produces three logical outputs:

- postings
- vocabulary
- document stats

The output formats are:

```text
postings: term<TAB>doc_id<TAB>tf<TAB>title<TAB>doc_length
vocabulary: term<TAB>df
documents: doc_id<TAB>title<TAB>doc_length
```

Where:

- `tf` = term frequency in one document
- `df` = document frequency in the whole corpus
- `doc_length` = token count for one document

The pipeline writes the HDFS folders:

- `/indexer/postings`
- `/indexer/vocabulary`
- `/indexer/documents`

## 6. MapReduce Pipeline 2

Pipeline 2 computes corpus-level statistics.

It reads the document stats from pipeline 1 and calculates:

- total document count `N`
- average document length `AVGDL`

The output format is:

```text
STAT<TAB>N<TAB>value
STAT<TAB>AVGDL<TAB>value
```

These values are stored in HDFS under:

- `/indexer/stats`

They are needed by the BM25 scoring formula.

## 7. Cassandra Schema

The Cassandra keyspace is `search_engine`.

The schema is created in [app/app.py](/Users/jeanne/code%20projects/BD_ass2_new/app/app.py).

Tables and data types:

```sql
CREATE KEYSPACE IF NOT EXISTS search_engine
WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};

CREATE TABLE documents (
    doc_id text PRIMARY KEY,
    title text,
    doc_length int
);

CREATE TABLE vocabulary (
    term text PRIMARY KEY,
    df int
);

CREATE TABLE postings (
    term text,
    doc_id text,
    tf int,
    title text,
    doc_length int,
    PRIMARY KEY (term, doc_id)
);

CREATE TABLE corpus_stats (
    stat_name text PRIMARY KEY,
    stat_value double
);
```

Example logical rows:

- `documents`: `("10174562", "A History of Money and Banking in the United States", 1234)`
- `vocabulary`: `("money", 58)`
- `postings`: `("money", "10174562", 4, "A History of Money and Banking in the United States", 1234)`
- `corpus_stats`: `("AVGDL", 842.61)`

The loader script truncates these tables on each run and reloads them from HDFS.

## 8. BM25 Search

The search logic is implemented in [app/query.py](/Users/jeanne/code%20projects/BD_ass2_new/app/query.py).

The BM25 parameters are:

- `k1 = 1.2`
- `b = 0.75`

The scoring formula used in [app/engine_utils.py](/Users/jeanne/code%20projects/BD_ass2_new/app/engine_utils.py) is:

```text
idf = log(N / df)
norm = k1 * ((1 - b) + b * (doc_length / AVGDL))
score = idf * ((tf * (k1 + 1)) / (tf + norm))
```

The query flow is:

1. Read the query text from command line arguments.
2. Tokenize the text with a simple alphanumeric regex.
3. Load `N` and `AVGDL` from `corpus_stats`.
4. For each query term, read `df` from `vocabulary`.
5. Read all matching postings from `postings`.
6. Calculate a BM25 score for each matching document-term pair.
7. Sum scores for the same document.
8. Sort the results.
9. Print the top 10 documents as `doc_id<TAB>title`.

The final search is run by [app/search.sh](/Users/jeanne/code%20projects/BD_ass2_new/app/search.sh).

It uses:

- `--master yarn`
- `--deploy-mode client`
- `--driver-memory 512m`
- `--executor-memory 512m`

The job runs successfully in YARN client mode and prints 10 ranked documents.

## 9. Technical Choices

I made a few choices to keep the project stable:

- I used exactly 100 documents.
- I used PySpark for preparation and search.
- I used Hadoop MapReduce for indexing.
- I used Cassandra for the final index.
- I used one shared Docker image for both Hadoop nodes.
- I moved Hadoop XML settings into mounted config files.
- I used one Spark archive in HDFS instead of many jar files, because it is more stable for YARN localization in Docker.

These choices do not change the assignment idea. They only make the environment easier to run and easier to grade.

## 10. Validation

I tested the project from a clean clone with:

```bash
docker compose up --build
```

The full run completes with:

- data preparation
- MapReduce indexing
- Cassandra loading
- BM25 search

The final log ends with:

- `Job 0 finished: takeOrdered ...`
- `SparkContext is stopping with exitCode 0.`
- `cluster-master exited with code 0`

This shows that the complete pipeline works from start to finish.

## 11. Screenshots

### 11.1 Indexing

Insert a screenshot that shows:

- `Prepared HDFS paths:`
- `Found 100 items`
- `Index created in HDFS:`
- `Index data loaded into Cassandra keyspace search_engine.`

### 11.2 Search

Insert a screenshot that shows:

- `Searching for documents using BM25`
- `Job 0 finished: takeOrdered ...`
- 10 result lines
- `SparkContext is stopping with exitCode 0.`

## 12. Conclusion

The project is a working mini search engine for 100 documents.

It uses:

- PySpark for data preparation
- Hadoop MapReduce for indexing
- Cassandra for storage
- BM25 for ranking

The final result is a reproducible Docker Compose project that starts from one command and finishes successfully.
