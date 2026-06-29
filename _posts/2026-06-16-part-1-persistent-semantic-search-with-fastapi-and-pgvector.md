---
title: 'Building a Production-Ready Food Nutrition Semantic Search and RAG System — Part 1: Persistent Semantic Search with FastAPI and pgvector'
date: 2026-06-16
collection: posts
permalink: /posts/2026/06/part-1-persistent-semantic-search-with-fastapi-and-pgvector/
toc: true
tags:
  - ciqual
  - semantic-search
  - postgresql
  - pgvector
  - fastapi
  - embeddings
---
This post marks the beginning of a series on **Building a production-ready food nutrition semantic search and Retrieval-Augmented Generation (RAG) system** from the ground up. Over the course of this series, we'll move from data engineering and semantic retrieval to LLM-powered question answering, APIs, user interfaces, and deployment.

Before starting this series, we laid the foundation in a [previous post](https://tantikristanti.github.io/posts/2026/06/learning-to-embrace-imperfection-building-resilient-etl-pipelines-with-ciqual/), where we built a resilient ETL pipeline to ingest the French CIQUAL food composition dataset into PostgreSQL. The result was a clean, validated relational database containing thousands of food and nutritional records.

However, a database alone is not enough. A static database is like a library without a librarian: the information exists, but users cannot easily ask natural-language questions such as *"What foods are rich in omega-3 but low in mercury?"* or *"Quels aliments contiennent beaucoup de protéines et peu de matières grasses ?"*

In this first installment, we'll transform that structured dataset into a semantic search engine by generating vector embeddings, storing them with pgvector, and exposing a search API with FastAPI. By the end, we'll have a RAG-ready retrieval layer capable of finding nutritionally relevant foods based on meaning rather than exact keyword matches; a critical building block for the intelligent food assistant we'll develop throughout the rest of this series.

---

## Introduction

Semantic search is not magic. It boils down to a simple idea: transform each document (food description) and the user's query into numerical vectors (embeddings) that capture meaning. Then find the documents whose vectors are "closest" to the query vector. The closer they are, the more relevant they are.

Many tutorials implement this with dedicated vector databases such as Qdrant, Pinecone, or Weaviate. Those are excellent tools, but they introduce separate infrastructure. If we are already running PostgreSQL, adding a vector extension keeps our architecture simpler and easier to maintain.

**pgvector** is an open‑source extension that turns PostgreSQL into a vector database. It adds a new `vector` data type, index types (IVFFlat, HNSW), and distance operators (`<->` for Euclidean, `<=>` for cosine). With `pgvector`, we can find vectors similar to the user's query using ordinary SQL:

```sql
SELECT * FROM foods ORDER BY embedding <=> query_embedding LIMIT 5;
```

In this blog, we will integrate `pgvector` into our CIQUAL pipeline, generate embeddings using a multilingual model (French and English work out‑of‑the‑box), and wrap everything in a FastAPI service. The result is a production‑ready semantic search engine that also serves as a perfect retrieval component for a RAG (Retrieval-Augmented Generation) system.

---

## Why Not Dedicated Vector Databases? (A Comparison)

If you have read my earlier post on semantic search with FastEmbed and Qdrant, you may wonder: why not use Qdrant again?

Both approaches are valid, but they serve different needs. The following table summarises the differences [[1][1], [2][2]]:

| Aspect                           | Dedicated (Qdrant)                                                                   | In‑database (pgvector)                           |
| -------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------- |
| Infrastructure                   | Additional service to maintain                                                       | No extra component                                |
| Metadata joins                   | Payloads stored inside Qdrant, but JOINs with external DBs require application logic | Natural SQL JOINs with any table                  |
| ACID transactions                | Not supported across DBs                                                             | Full ACID with the rest of our data               |
| Scaling                          | Designed for billions of vectors                                                     | Good up to millions, then needs careful tuning    |
| Hybrid search (vector + keyword) | Built‑in                                                                            | Requires pg_trgm or tsvector combined with vector |
| Ease of use                      | REST/gRPC API, client libraries                                                      | Ordinary SQL, any PostgreSQL driver               |

For our CIQUAL dataset (~3,500 foods) [[3][3]], a dedicated vector database is overkill. `pgvector` integrates seamlessly into the existing ETL pipeline and keeps the stack lean.

---

## Step 1: Enable `pgvector` in PostgreSQL

If you used the Docker [[4][4]] command from [the previous ETL blog](https://tantikristanti.github.io/posts/2026/06/learning-to-embrace-imperfection-building-resilient-etl-pipelines-with-ciqual/) to run Postgres [[5][5]] container, you already have the `pgvector/pgvector` image [[6][6]]. If not, start the container with:

```bash
docker run -d --name pgvector \
  -e POSTGRES_USER=ciqual \
  -e POSTGRES_PASSWORD=ciqual \
  -e POSTGRES_DB=ciqual_db \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

Connect and enable the extension:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Verify:

```sql
SELECT * FROM pg_extension;
```

Now our PostgreSQL can store and search vectors.

---

## Step 2: Choose a Multilingual Embedding Model

We need a model that understands both French and English, because CIQUAL provides names in both languages and users may ask in either. In this project, we will use **`paraphrase-multilingual-MiniLM-L12-v2`** from Sentence‑Transformers [[7][7]].

Why this model?

* **384‑dimension vectors** – small enough to be efficient, large enough to capture meaning.
* **Multilingual** – supports 50+ languages, including French and English.
* **Fast** – runs on CPU with reasonable latency (~30ms per query).
* **Apache 2.0 license** – free for commercial use.

The model is loaded once and reused for both generating food embeddings and encoding queries.

---

## Step 3: Create the Embeddings Table

Inside our [`embedding_generator.py` module](https://github.com/tantikristanti/Generative-AI-LLMs/blob/main/food-nutrition-semantic-search-rag/src/ciqual_etl/embedding_generator.py), we define a new table:

```sql
CREATE TABLE IF NOT EXISTS food_composition_embeddings (
    id SERIAL PRIMARY KEY,
    alim_code INTEGER,
    alim_nom_fr TEXT,
    alim_nom_eng TEXT,
    composition_text TEXT,
    metadata JSONB,
    embedding vector(384)
);
```

* `composition_text` is the human‑readable description we will embed.
* `metadata` stores the original row as JSONB.
* `embedding` is the pgvector column.

We also create an **IVFFlat index** on the embedding column to speed up cosine similarity searches:

```sql
CREATE INDEX IF NOT EXISTS idx_food_embeddings
ON food_composition_embeddings
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

IVFFlat is an approximate nearest neighbour index. With `lists = 100`, the search will be accurate and fast for up to tens of thousands of vectors.

---

## Step 4: Build Text Representations for Each Food

The embedding model requires text to embed. Therefore, we convert each food's row into a readable string that includes its name, group, and nutrient values. For example:

```text
alim_grp_code: 4
alim_ssgrp_code: 409
alim_ssssgrp_code: 0
alim_grp_nom_eng: meat, egg and fish
alim_ssgrp_nom_eng: fish products
alim_code: 25433
alim_nom_eng: Caribbean-style fish fritters, prepacked
alim_nom_fr: Accra de poisson, préemballé
Energy (kJ 100g): 1070
Protein (g 100g): 8.51
Fat (g 100g): 14.8
...
```

The function `_row_to_text()` loops over every column and builds this string. Missing values become "N/A". This text is later used both for embedding and for returning readable context to the user.

---

## Step 5: Generate and Store Embeddings

The core of the `FoodEmbeddingGenerator` class is the `run()` method:

```python
def run(self, drop_existing: bool = False):
    df = pd.read_csv(self.csv_path, sep='\t')
    texts = []
    metadata_list = []
    for _, row in df.iterrows():
        texts.append(self._row_to_text(row))
        metadata_list.append({
            "alim_code": row.get('alim_code'),
            "alim_nom_fr": row.get('alim_nom_fr'),
            "alim_nom_eng": row.get('alim_nom_eng'),
            "row_data": row.to_dict()
        })

    embeddings = self._generate_embeddings(texts)  # batch encoding
    vector_dim = embeddings.shape[1]

    self._connect_db()
    try:
        if drop_existing:
            self.cur.execute("DROP TABLE IF EXISTS food_composition_embeddings CASCADE;")
        self._create_table(vector_dim)

        data = []
        for idx, row in df.iterrows():
            emb_vec = '[' + ','.join(str(x) for x in embeddings[idx]) + ']'
            data.append((row['alim_code'], row['alim_nom_fr'], row['alim_nom_eng'],
                         texts[idx], json.dumps(metadata_list[idx]), emb_vec))

        execute_values(self.cur, """
            INSERT INTO food_composition_embeddings
            (alim_code, alim_nom_fr, alim_nom_eng, composition_text, metadata, embedding)
            VALUES %s
        """, data)
        self.conn.commit()
    finally:
        self._disconnect_db()
```

Running this script once populates the embeddings table.

---

## Step 6: The Search Retriever Class

Now that embeddings are stored, we need a way to query them. The `FoodRetriever` class in [`food_search_engine.py`](https://github.com/tantikristanti/Generative-AI-LLMs/blob/main/food-nutrition-semantic-search-rag/src/ciqual_etl/food_search_engine.py) does exactly that.

```python
class FoodRetriever:
    def __init__(self, model_name: str = "paraphrase-multilingual-MiniLM-L12-v2"):
        self.model = SentenceTransformer(model_name)
        self._connect_db()

    def search(self, query: str, top_k: int = 5):
        query_emb = self.model.encode([query])[0]
        query_vec = '[' + ','.join(str(x) for x in query_emb) + ']'

        sql = """
            SELECT
                alim_code, alim_nom_fr, alim_nom_eng,
                composition_text, metadata,
                1 - (embedding <=> %s::vector) AS similarity
            FROM food_composition_embeddings
            ORDER BY embedding <=> %s::vector
            LIMIT %s;
        """
        self.cur.execute(sql, (query_vec, query_vec, top_k))
        rows = self.cur.fetchall()
        results = []
        for row in rows:
            results.append({
                "alim_code": row[0],
                "alim_nom_fr": row[1],
                "alim_nom_eng": row[2],
                "composition_text": row[3],
                "metadata": row[4],
                "similarity": round(row[5], 4)
            })
        return results
```

The operator `<=>` computes cosine distance. We transform it to similarity with `1 - distance`. The index we created earlier ensures this query runs in milliseconds.

---

## Step 7: FastAPI Wrapper – Expose the Search as a REST API

To make the search engine usable by other applications (web frontend, mobile app, RAG pipeline), we wrap it in a FastAPI application [[8][8]]. The [`fastapi_app.py`](https://github.com/tantikristanti/Generative-AI-LLMs/blob/main/food-nutrition-semantic-search-rag/src/ciqual_etl/fastapi_app.py) module contains:

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel
from contextlib import asynccontextmanager
from .food_search_engine import FoodRetriever

_retriever = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global _retriever
    _retriever = FoodRetriever()
    yield
    _retriever.close()

app = FastAPI(lifespan=lifespan)

class SearchRequest(BaseModel):
    query: str
    top_k: int = 5

class SearchResultItem(BaseModel):
    alim_code: int
    alim_nom_fr: str
    alim_nom_eng: str
    composition_text: str
    metadata: dict
    similarity: float

class SearchResponse(BaseModel):
    query: str
    top_k: int
    results: List[SearchResultItem]

@app.post("/search", response_model=SearchResponse)
async def search_post(request: SearchRequest):
    results = _retriever.search(request.query, request.top_k)
    return SearchResponse(query=request.query, top_k=request.top_k, results=results)

@app.get("/search", response_model=SearchResponse)
async def search_get(query: str = Query(...), top_k: int = 5):
    results = _retriever.search(query, top_k)
    return SearchResponse(query=query, top_k=top_k, results=results)
```

The application starts with the `uvicorn` [[9][9]] command:

```bash
uv run uvicorn ciqual_etl.fastapi_app:app --reload --port 8000
```

Interactive API documentation is available at `http://localhost:8000/docs`.

---

## Step 8: Testing the API (with Postman or curl)

**GET example** (URL parameters, [Figure 1](#fig1)):

```bash
curl "http://localhost:8000/search?query=poisson%20gras%20om%C3%A9ga%203&top_k=3"
```

<figure id="fig1">
  <img src="/images/posts/2026-06-16-part-1-persistent-semantic-search-with-fastapi-and-pgvector/1-postman-query-get.png" alt="query-get" height="100%" weight="100%">
  <figcaption>Figure 1: Search Endpoint (GET).</figcaption>
</figure>

**POST example** (JSON body, [Figure 2](#fig2)):

```bash
curl -X POST "http://localhost:8000/search" \
  -H "Content-Type: application/json" \
  -d '{"query": "low carbohydrate vegetables", "top_k": 5}'
```

<figure id="fig2">
  <img src="/images/posts/2026-06-16-part-1-persistent-semantic-search-with-fastapi-and-pgvector/2-postman-query-post.png" alt="query-post" height="100%" weight="100%">
  <figcaption>Figure 2: Search Endpoint (POST).</figcaption>
</figure>

Both return the following response:

```json
{
  "query": "poisson gras oméga 3",
  "top_k": 3,
  "results": [
    {
      "alim_code": 25433,
      "alim_nom_fr": "Accra de poisson, préemballé",
      "similarity": 0.613,
      "composition_text": "alim_grp_code: 4\n...",
      "metadata": {"alim_code": 25433, ...}
    }
  ]
}
```

---

## Performance and Scaling Considerations

With 3,500 foods, the search latency is consistently below 100 ms on a modest laptop [[10][10]]. The IVFFlat index with `lists = 100` provides a good balance, recall is ~98% compared to exact brute‑force [[10][10], [11][11]].

If your dataset grows to hundreds of thousands or millions of rows, consider:

* Increasing `lists` to `sqrt(number_of_rows)`.
* Switching to the **HNSW** index (`USING hnsw`), which offers higher recall but slower index build.
* Partitioning the table by food category.
* Using a dedicated vector database if you exceed 10 million vectors.

For our use case, `pgvector` is more than enough.

---

## Conclusion

In this first installment of our **Building a Production-Ready Food Nutrition Semantic Search and RAG System** series, we transformed the CIQUAL nutritional database from a static repository of food data into a multilingual semantic search engine.

Using PostgreSQL and the `pgvector` extension, we generated and stored vector embeddings alongside the original food metadata, enabling natural-language retrieval in both French and English. We then exposed this functionality through a FastAPI application, providing a clean and documented REST API that can retrieve nutritionally relevant foods based on meaning rather than exact keyword matches.

More importantly, we've established the retrieval foundation that every RAG system depends on. The API not only returns the most relevant foods but also includes rich nutritional metadata, making it ready to serve as the knowledge retrieval layer for downstream AI applications.

### Building Toward a Complete RAG System

At the beginning of this article, we compared a traditional database to a library without a librarian. With semantic search in place, we've effectively built the librarian: users can now search the food knowledge base using natural language and receive contextually relevant results.

However, retrieval is only the first half of the journey.

A complete Retrieval-Augmented Generation (RAG) system combines:

1. **Retrieval** – finding the most relevant information from a knowledge base.
2. **Generation** – using a large language model to transform that information into a natural-language answer.

This article focused on the retrieval layer. In the next installment, we'll build the generation layer by integrating the search engine with Ollama and a local LLM, allowing users to ask complex nutrition questions and receive conversational, evidence-based answers grounded in the CIQUAL dataset.

By the end of the series, we'll have progressed from raw nutritional data to a fully functional, production-ready food nutrition assistant powered by semantic search and RAG.

---

### What's Next?

*This post is part of the series **Building a Production-Ready Food Nutrition Semantic Search and RAG System**.*

* ***[Part 1: Persistent Semantic Search with FastAPI and pgvector](https://tantikristanti.github.io/posts/2026/06/part-1-persistent-semantic-search-with-fastapi-and-pgvector/)** (this post)*
* *Part 2: Designing a Modular Food RAG Architecture with Object-Oriented Principles*
* *Part 3: Exposing the RAG System with a FastAPI Backend*
* *Part 4: From Fixed Pipeline to Intelligent Multi-Agent RAG*

In **Part 2**, we'll design a modular food nutrition RAG architecture with Object-Oriented Principles.

---

### 📦 GitHub Repository

The complete, runnable code for the ETL pipeline, embedding generation, semantic search engine, FastAPI server, and supporting components is available in the GitHub repository under the `ciqual_etl` package.

Repository: [https://github.com/tantikristanti/Generative-AI-LLMs/tree/main/food-nutrition-semantic-search-rag](https://github.com/tantikristanti/Generative-AI-LLMs/tree/main/food-nutrition-semantic-search-rag)

---

## References

1. Ivan Cernja. (2026). **pgvector vs Qdrant in 2026**. [https://encore.dev/articles/pgvector-vs-qdrant](https://encore.dev/articles/pgvector-vs-qdrant).
2. Alibaba Cloud. (2026). **SQL references**. [https://www.alibabacloud.com/help/en/polardb/polardb-for-postgresql/pgvector-sql-reference/](https://www.alibabacloud.com/help/en/polardb/polardb-for-postgresql/pgvector-sql-reference/).
3. ANSES. (2025).  **Ciqual French food composition table** . [https://ciqual.anses.fr/](https://ciqual.anses.fr/)
4. Docker Inc. (2026). **Docker Documentation**. [https://docs.docker.com/](https://docs.docker.com/).
5. The PostgreSQL Global Development Group. (2026). **PostgreSQL: The World&#39;s Most Advanced Open Source Relational Database**. [https://www.postgresql.org/](https://www.postgresql.org/).
6. pgvector. (2026).  **pgvector: Open‑source vector similarity search for PostgreSQL** . [https://github.com/pgvector/pgvector](https://github.com/pgvector/pgvector).
7. Sentence‑Transformers. (2026).  **paraphrase-multilingual-MiniLM-L12-v2** . [https://huggingface.co/sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2](https://huggingface.co/sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2).
8. FastAPI. (2026).  **FastAPI framework** . [https://fastapi.tiangolo.com/](https://fastapi.tiangolo.com/).
9. Uvicorn. (2026).  **Uvicorn ASGI server** . [https://uvicorn.dev/](https://uvicorn.dev/).
10. Mark AI. (2026). **pgvector HNSW vs IVFFlat: Latency Benchmarks on 500K Vectors**. [https://markaicode.com/benchmarks/postgresql-pgvector-benchmark/](https://markaicode.com/benchmarks/postgresql-pgvector-benchmark/)
11. Mastra AI. (2025). **Benchmarking pgvector RAG performance across different dataset sizes**. [https://mastra.ai/blog/pgvector-perf](https://mastra.ai/blog/pgvector-perf).

---

[1]: https://encore.dev/articles/pgvector-vs-qdrant
[2]: https://www.alibabacloud.com/help/en/polardb/polardb-for-postgresql/pgvector-sql-reference/
[3]: https://ciqual.anses.fr/#/cms/download/node/20
[4]: https://docs.docker.com/
[5]: https://www.postgresql.org/
[6]: https://github.com/pgvector/pgvector
[7]: https://huggingface.co/sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2
[8]: https://fastapi.tiangolo.com/
[9]: https://uvicorn.dev/
[10]: https://markaicode.com/benchmarks/postgresql-pgvector-benchmark/
[11]: https://mastra.ai/blog/pgvector-perf
