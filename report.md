# Big Data Assignment 2 — Report

---

## 1. Methodology

### 1.1 System Overview

The goal of this assignment was to build a distributed search engine for plain text documents. The system takes a corpus of 100 documents, indexes them using Hadoop MapReduce, stores the resulting index in Cassandra, and answers free-text queries using BM25 ranking. The full pipeline runs inside Docker Compose and requires no manual setup steps beyond `docker compose up`.

The pipeline uses distributed tools at every major stage: PySpark for data preparation and query execution, Hadoop MapReduce for index construction, and Cassandra as the storage and retrieval layer. No single-machine libraries such as pandas were used at any point.

---

### 1.2 Cluster Architecture

The cluster is defined with Docker Compose and consists of three containers:

- **`cluster-master`** — orchestrates the full workflow: starts Hadoop services, prepares the data with PySpark, launches MapReduce jobs, loads data into Cassandra, and submits the search job to YARN.
- **`cluster-slave-1`** — acts as the Hadoop worker node, running YARN NodeManager and HDFS DataNode. MapReduce tasks and YARN executors are scheduled here.
- **`cassandra-server`** — stores the final inverted index and serves low-latency lookups during query time.

Both `cluster-master` and `cluster-slave-1` use the same Docker image. This ensures that the Python runtime and all dependencies are identical on both nodes, which is important for YARN-distributed PySpark execution.

---

### 1.3 Data Preparation (`prepare_data.py`)

Data preparation is implemented as a PySpark job. The primary data source is a Parquet file at `/a.parquet` in HDFS, containing three columns: `id`, `title`, and `text`. If that file is not found, the pipeline falls back to bundled plain text files in `app/data`.

PySpark reads the source, drops rows where `text` is null or empty, and samples exactly 100 documents. For each selected document, the script generates a local `.txt` file named in the format `<doc_id>_<doc_title>.txt`. These files are uploaded to HDFS under `/data/`, then read back with Spark and written to `/input/data` as a single-partition text file in tab-separated format:

```
doc_id<TAB>title<TAB>text
```

This file is the input for the first MapReduce pipeline.

**Design note:** this stage is a hybrid preprocessing step. PySpark is used for all reading, filtering, and sampling — no pandas or single-machine data processing libraries are involved. However, Spark runs in local mode here, and the driver writes local `.txt` files before uploading them to HDFS. This approach was chosen for simplicity and stability in the Docker environment and does not affect the correctness of the downstream distributed pipeline.

---

### 1.4 Index Construction (`create_index.sh`)

The index is built with two sequential MapReduce jobs.

#### Pipeline 1 — Inverted Index (`mapper1.py` / `reducer1.py`)

The mapper reads each line of `/input/data`, tokenizes the document text using alphanumeric tokens, and emits two types of records:

- A **document metadata record** per document:
  ```
  __DOC__{doc_id}    DOC    {doc_id}    {title}    {doc_length}
  ```
  where `doc_length` is the total token count.

- A **posting record** per distinct term:
  ```
  {term}    POSTING    {doc_id}    {tf}    {title}    {doc_length}
  ```
  where `tf` is term frequency within the document.

The reducer groups records by key. For each term it emits one vocabulary row (`VOCAB    {term}    {df}`) and one posting row per matching document. Document metadata rows are passed through unchanged.

After the MapReduce job completes, `create_index.sh` splits the output into three separate TSV files in HDFS:

| HDFS path | Content |
|---|---|
| `/indexer/postings` | `term, doc_id, tf, title, doc_length` |
| `/indexer/vocabulary` | `term, df` |
| `/indexer/documents` | `doc_id, title, doc_length` |

#### Pipeline 2 — Corpus Statistics (`mapper2.py` / `reducer2.py`)

The second job reads the `DOC` records produced by Pipeline 1. Each document contributes `1` to the document count and its `doc_length` to the total token count. The reducer computes:

- `N` — total number of documents
- `AVGDL` — average document length in tokens

Results are stored at `/indexer/stats` in the format:
```
STAT    N       100
STAT    AVGDL   <value computed from the indexed corpus>
```

These values are required by the BM25 scoring formula.

---

### 1.5 Cassandra Storage (`app.py`, `store_index.sh`)

Once the HDFS index is ready, all four datasets are loaded into Cassandra under the keyspace `search_engine` (SimpleStrategy, replication factor 1). The loader script truncates all tables on each run and reloads them from HDFS, which makes the pipeline idempotent.

The schema is as follows:

```sql
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

The `postings` table uses a composite primary key `(term, doc_id)`, which allows Cassandra to efficiently retrieve all documents containing a given term — exactly the access pattern needed at query time.

Example rows from a sample run:

- `documents`: `("10174562", "A History of Money and Banking in the United States", 1234)`
- `vocabulary`: `("money", 58)`
- `postings`: `("money", "10174562", 4, "A History of Money and Banking in the United States", 1234)`
- `corpus_stats`: `("AVGDL", 842.61)` *(example value from one run)*

---

### 1.6 BM25 Search (`query.py`, `search.sh`)

Search queries are executed by a PySpark job submitted to YARN in client mode. The job connects to Cassandra, retrieves the relevant index data, and ranks documents using the BM25 formula.

**BM25 parameters** (defined in `engine_utils.py`):
- `k1 = 1.2`
- `b = 0.75`

**Scoring formula** for each (term, document) pair:

```
idf  = log(N / df)
norm = k1 * ((1 - b) + b * (doc_length / AVGDL))
score = idf * (tf * (k1 + 1)) / (tf + norm)
```

**Query flow:**

1. The query string is taken from the command line.
2. It is tokenized with the regex `[A-Za-z0-9]+` and lowercased.
3. The app connects to `cassandra-server` and reads `N` and `AVGDL` from `corpus_stats`.
4. For each query term, `df` is looked up in `vocabulary` and all matching rows are fetched from `postings`.
5. Each row becomes a tuple `(doc_id, title, tf, doc_length, df)`, loaded into a Spark RDD.
6. BM25 scores are computed in parallel; `reduceByKey` sums scores across terms for each document.
7. `takeOrdered(10, key=lambda x: -x[2])` returns the top 10 results.
8. Results are printed as `doc_id<TAB>title`.

The `search.sh` script submits the job to YARN in client mode:

```bash
spark-submit \
  --master yarn \
  --deploy-mode client \
  --driver-memory 512m \
  --executor-memory 512m \
  --py-files /app/engine_utils.py \
  query.py "$@"
```

Client mode was chosen over cluster mode because it keeps the driver on the master node, which simplifies log access and makes the pipeline easier to monitor and grade in this Docker environment. The executors still run on the YARN cluster, so the execution remains distributed.

---

## 2. Demonstration

### 2.1 How to Run

Make sure Docker Desktop is running. Clone the repository and start the full pipeline with:

```bash
docker compose up --build
```

The master container will automatically run every step in sequence:

1. Start Hadoop and Cassandra services
2. Prepare 100 documents and upload them to HDFS
3. Build the inverted index with two MapReduce jobs
4. Load the index into Cassandra
5. Execute sample BM25 search queries and print results

No manual steps are required. The pipeline finishes when you see:

```
SparkContext is stopping with exitCode 0.
cluster-master exited with code 0
```

To run a custom query after the pipeline has finished:

```bash
docker exec cluster-master bash /app/search.sh "your query here"
```

---

### 2.2 Screenshot 1 — Successful Indexing of 100 Documents

![alt text](<Снимок экрана 2026-03-31 в 19.33.56.png>)
The screenshot shows the log lines confirming that all indexing stages completed successfully, including `Prepared HDFS paths`, `Found 100 items`, `Index created in HDFS`, and `Index data loaded into Cassandra keyspace search_engine`.

---

### 2.3 Screenshot 2 — Search Query 1 - "history money banking"
![alt text](<Снимок экрана 2026-04-03 в 17.14.15.png>)

---

### 2.4 Screenshot 3 — Search Query 2 - "sherlock holmes"

![alt text](<Снимок экрана 2026-04-03 в 17.15.14.png>)

---

### 2.5 Screenshot 4 — Search Query 3 - "crime drama film"

![alt text](<Снимок экрана 2026-04-03 в 17.22.00.png>)
---

### 2.6 Reflections on Results

The BM25 ranking behaves as expected across all tested queries. Documents containing multiple query terms with high term frequency rank at the top, while documents with only partial overlap appear lower in the list.

The `idf` component of the formula means that terms appearing in many of the 100 documents contribute less to the final score, while rare and specific terms have stronger discriminative power. This is visible in practice: a query like `"money banking"` returns documents specifically about finance and economics rather than general articles that happen to mention these words once.

The `b = 0.75` length normalization ensures that longer documents are not unfairly rewarded for simply containing more tokens. With an average document length of several hundred tokens across the corpus, this normalization has a noticeable effect on ranking order.

One limitation of the current setup is the small corpus size. With only 100 documents, some queries return fewer than 10 strongly relevant results, so the lower-ranked entries may have only marginal term overlap with the query. A larger corpus would produce sharper distinctions between relevant and non-relevant documents and make the BM25 scores more meaningful.

---

## 3. Conclusion

The assignment is complete. The project implements a full distributed search pipeline:

- **Data preparation** with PySpark, reading from a Parquet corpus and writing to HDFS
- **Inverted index construction** with two Hadoop MapReduce jobs, producing postings, vocabulary, document metadata, and corpus statistics
- **Index storage** in Cassandra, with a schema designed around the access patterns of BM25 retrieval
- **Query execution** with PySpark on YARN, using BM25 ranking to return the top 10 documents per query

The system starts with `docker compose up` and requires no manual steps. The output is reproducible and confirmed across a clean clone and full Docker run.
