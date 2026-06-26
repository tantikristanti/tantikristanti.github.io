---
title: 'Building a Production-Ready Food Nutrition Semantic Search and RAG System — Part 3: Exposing the RAG System with a FastAPI Backend'
date: 2026-06-21
collection: posts
permalink: /posts/2026/06/part-3-exposing-the-rag-system-with-a-fastapi-backend/
toc: true
tags:
  - rag
  - fastapi
  - modular-architecture
  - semantic-search
---
This is **Part 3** of the series **Building a Production‑Ready Food Nutrition Semantic Search and RAG System**. If you're new to the series, start with [Part 1: Persistent Semantic Search with FastAPI and pgvector](https://tantikristanti.github.io/posts/2026/06/part-1-persistent-semantic-search-with-fastapi-and-pgvector/) and [Part 2: Designing a Modular Food RAG Architecture with Object‑Oriented Principles](https://tantikristanti.github.io/posts/2026/06/part-2-designing-a-modular-food-rag-architecture-with-object-oriented-principles/), where we built the multilingual retrieval engine and the modular RAG core that power this system.

---

In [Part 2](https://tantikristanti.github.io/posts/2026/06/part-2-designing-a-modular-food-rag-architecture-with-object-oriented-principles/), we built a complete RAG system using Object‑Oriented Programming principles. We designed abstract base classes, concrete implementations for retrieval, prompt building, and LLM communication, and an orchestrator (`RAGSystem`) that composes them together. The result was a clean, testable, and extensible core.

But a core library isn't useful on its own. It needs an interface. In this post, we'll build a **production‑ready FastAPI backend** that wraps our RAG system and exposes it over HTTP. We'll add:

* **REST API endpoints** for text queries, multimodal (image + text) queries, image generation, and streaming responses.
* **Lazy loading** of the RAG system for fast startup.
* **Health checks** with database connectivity validation.
* **File upload handling** for images and returning generated image files.
* **Temporary file cleanup** using FastAPI's `BackgroundTasks`.
* **Centralised configuration** with Pydantic settings.
* **Comprehensive error handling** and structured responses.
* **Testing instructions** with `curl` and Postman.

By the end of this post, you'll have a deployable API that can be consumed by web frontends, mobile apps, or other microservices.

---

## Why a FastAPI Backend?

FastAPI is the natural choice for this task:

* **High performance** — on par with Node.js and Go.
* **Automatic OpenAPI documentation** — interactive `/docs` endpoint.
* **Type safety** with Pydantic models.
* **Async support** for non‑blocking I/O.
* **Built‑in validation** and serialisation.

In our case, FastAPI provides a clean separation between the HTTP layer and our core RAG logic. The API endpoints handle request validation, file parsing, and response formatting, while delegating the heavy lifting to the `RAGSystem` we built in Part 2.

---

## Project Structure

Here's where we are in the project:

```text
food-nutrition-semantic-search-rag/
└── src/
    └── rag/
        ├── __init__.py              # Package exports (lazy loading)
        ├── config.py                # Pydantic settings
        ├── food_image_retriever.py  # PostgreSQL + pgvector retriever
        ├── prompt_builder.py        # Prompt formatting logic
        ├── ollama_client.py         # Ollama HTTP client
        ├── rag_base.py              # Abstract base classes
        ├── rag_system.py            # The orchestrator (RAGSystem)
        └── rag_fastapi_app.py       # FastAPI application (this post)
```

The `rag_fastapi_app.py` file is our entry point. It imports the `RAGSystem` from the `rag` package, defines endpoints, and runs the server.

---

## 1. Designing the FastAPI Endpoints

We'll expose four main endpoints:

| Endpoint                | Method   | Description                                    |
| ----------------------- | -------- | ---------------------------------------------- |
| `/health`             | `GET`  | Health check with database connectivity status |
| `/rag/query`          | `POST` | Text‑only RAG query                           |
| `/rag/multimodal`     | `POST` | RAG query with text + image (multipart form)   |
| `/rag/generate-image` | `POST` | Generate a food image from a text description  |
| `/rag/stream`         | `POST` | Stream RAG response token by token             |

Let's walk through each endpoint implementation.

### Lazy Loading the RAG System

We want the server to start quickly, even if the RAG system is heavy to initialise. We use a **lazy loading** pattern: the `RAGSystem` is only created when the first request arrives.

```python
# src/rag/rag_fastapi_app.py
import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI, UploadFile, File, Form, HTTPException, BackgroundTasks
from fastapi.responses import StreamingResponse, FileResponse
from pydantic import BaseModel
from typing import List, Optional
from PIL import Image
import io
import tempfile
import os

from rag import RAGSystem

logger = logging.getLogger(__name__)

# Global RAG instance (lazy loaded)
_rag_system = None

def get_rag_system() -> RAGSystem:
    """Lazy load RAG system — only created on first call."""
    global _rag_system
    if _rag_system is None:
        _rag_system = RAGSystem()
    return _rag_system
```

This approach is simple yet effective. Because the `RAGSystem` constructor initializes the embedding model and establishes the database connection, delaying its creation until the first request helps minimize server startup time.

### Database Connectivity Check

We also need a helper function to verify that the database is reachable. To do this, we perform a minimal search (with `top_k=1`), which forces the retriever to establish a database connection and execute a real SQL query.

```python
def _check_db_connection(rag: RAGSystem) -> tuple[bool, str]:
    """
    Check database connectivity by performing a minimal search.
    Returns (is_ok, message).
    """
    try:
        rag.retriever.search("test", top_k=1)
        return True, "connected"
    except Exception as e:
        return False, f"Database connection failed: {e}"
```

### Lifespan Management

We use FastAPI's `lifespan` context manager to manage application startup and shutdown events. During startup, we verify database connectivity, and during shutdown, we release any allocated resources.

```python
def _close_rag_system(rag: RAGSystem):
    """Safely close RAG system resources."""
    if rag is None:
        return

    if hasattr(rag, 'close') and callable(rag.close):
        rag.close()
        logger.info("Closed RAG System.")
        return

    if hasattr(rag, 'retriever') and hasattr(rag.retriever, 'close'):
        rag.retriever.close()
        logger.info("Closed FoodRetriever database connection.")
        return

    logger.warning("No close method found. Skipping explicit cleanup.")

@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("Starting up RAG system...")
    rag = get_rag_system()
    ok, msg = _check_db_connection(rag)
    if ok:
        logger.info("Database connection established successfully.")
    else:
        logger.error(f"Database connection failed on startup: {msg}")
    yield
    _close_rag_system(_rag_system)

app = FastAPI(
    title="Food Nutrition RAG API",
    version="1.0.0",
    lifespan=lifespan
)
```

### Health Check Endpoint

The `/health` endpoint reports the application's health status, including database connectivity.

```python
@app.get("/health")
async def health_check():
    """Health check with database connectivity status."""
    rag = get_rag_system()
    ok, msg = _check_db_connection(rag)
    return {
        "status": "ok" if ok else "degraded",
        "database": msg
    }
```

Example response:

```json
{
  "status": "ok",
  "database": "connected"
}
```

### Text Query Endpoint (`/rag/query`)

This endpoint accepts a JSON request containing the user's query along with optional configuration parameters.

```python
class QueryRequest(BaseModel):
    query: str
    top_k: int = 5
    model: Optional[str] = None
    temperature: float = 0.7

class QueryResponse(BaseModel):
    query: str
    answer: str
    documents: List[dict]
    model: str

@app.post("/rag/query")
async def rag_query(request: QueryRequest):
    """Query the RAG system with text."""
    rag = get_rag_system()
    response = rag.query(
        question=request.query,
        top_k=request.top_k,
        **{"model": request.model, "temperature": request.temperature}
    )

    return QueryResponse(
        query=response.query,
        answer=response.llm_response,
        documents=[{
            "food": doc.metadata.get("alim_nom_fr", "Unknown"),
            "score": doc.score,
            "content": doc.content[:500] + "...",
            "image_url": doc.image_url
        } for doc in response.retrieved_documents],
        model=response.model_used
    )
```

### Multimodal Query Endpoint (`/rag/multimodal`)

This endpoint accepts a text query and an image file as multipart form data. It validates and loads the image before passing both inputs to the RAG system's `query_multimodal` method.

```python
@app.post("/rag/multimodal")
async def rag_multimodal(
    query: str = Form(...),
    image: UploadFile = File(...),
    top_k: int = Form(5),
    model: Optional[str] = Form(None),
    temperature: float = Form(0.7)
):
    """Query the RAG system with text and an image."""
    if not image.content_type.startswith("image/"):
        raise HTTPException(400, "File must be an image")

    img_bytes = await image.read()
    img = Image.open(io.BytesIO(img_bytes))

    rag = get_rag_system()
    response = rag.query_multimodal(
        question=query,
        images=[img],
        top_k=top_k,
        **{"model": model, "temperature": temperature}
    )

    return QueryResponse(
        query=response.query,
        answer=response.llm_response,
        documents=[{
            "food": doc.metadata.get("alim_nom_fr", "Unknown"),
            "score": doc.score,
            "content": doc.content[:500] + "...",
            "image_url": doc.image_url
        } for doc in response.retrieved_documents],
        model=response.model_used
    )
```

### Image Generation Endpoint (`/rag/generate-image`)

This endpoint generates food images from text descriptions using Ollama's vision model. It supports two output modes:

- **Download mode** (default) — generates the image and returns it as a downloadable PNG file. Any temporary files created during the process are automatically removed after the response is sent.
- **Save mode** (optional) — when a `save_path` is provided, the image is saved to the specified location on the server and a JSON response is returned to confirm the operation.

```python
@app.post("/rag/generate-image")
async def generate_food_image(
    description: str = Form(...),
    model: Optional[str] = Form(None),
    save_path: Optional[str] = Form(None),
    background_tasks: BackgroundTasks = None
):
    """
    Generate a food image from text description.

    - If `save_path` is provided, saves to that location and returns JSON.
    - Otherwise, returns the image as a downloadable PNG file.
    """
    rag = get_rag_system()

    # If a save_path is specified, save directly and return JSON
    if save_path:
        if ".." in save_path:
            raise HTTPException(400, "Invalid save_path: path traversal not allowed.")

        dirname = os.path.dirname(save_path)
        if dirname and not os.path.exists(dirname):
            os.makedirs(dirname, exist_ok=True)

        try:
            rag.generate_food_image(description, out_file=save_path, model=model)
            return {
                "status": "success",
                "message": f"Image saved to {save_path}",
                "path": save_path
            }
        except Exception as e:
            raise HTTPException(500, f"Image generation failed: {str(e)}")

    # Default: return as downloadable file (temporary)
    with tempfile.NamedTemporaryFile(suffix=".png", delete=False) as tmp_file:
        tmp_path = tmp_file.name

    try:
        rag.generate_food_image(description, out_file=tmp_path, model=model)
        background_tasks.add_task(os.unlink, tmp_path)
        return FileResponse(tmp_path, media_type="image/png", filename="food_image.png")
    except Exception as e:
        if os.path.exists(tmp_path):
            os.unlink(tmp_path)
        raise HTTPException(500, f"Image generation failed: {str(e)}")
```

### Streaming Endpoint (`/rag/stream`)

This endpoint streams the LLM's output incrementally using FastAPI's `StreamingResponse`. Instead of waiting for the complete response to be generated, clients receive tokens as they become available, enabling a more responsive user experience. This is useful for chat‑style interfaces where users want to see the response as it's generated.

```python
@app.post("/rag/stream")
async def rag_stream(request: QueryRequest):
    """Stream RAG response token by token."""
    rag = get_rag_system()

    def generate():
        for token in rag.stream_query(
            question=request.query,
            top_k=request.top_k,
            **{"model": request.model, "temperature": request.temperature}
        ):
            yield token

    return StreamingResponse(generate(), media_type="text/plain")
```

---

## 2. Configuration Management with Pydantic Settings

Centralizing configuration improves maintainability and simplifies deployment. We achieve this using a `RAGConfig` dataclass, or alternatively, Pydantic's `BaseSettings` when environment variable support is required.

```python
# src/rag/config.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class RAGConfig:
    """Centralised configuration for the RAG system."""
    # LLM settings
    llm_model: str = "llama3.2"
    llm_base_url: str = "http://localhost:11434"
    temperature: float = 0.7

    # Retrieval settings
    default_top_k: int = 5
    embedding_model: str = "paraphrase-multilingual-MiniLM-L12-v2"

    # Database settings
    db_host: str = "localhost"
    db_port: int = 5432
    db_name: str = "food_nutrition"
    db_user: str = "postgres"
    db_password: str = ""

    # System prompt
    system_prompt: str = "You are a helpful nutrition assistant."
```

To use environment variables, you can switch to Pydantic's `BaseSettings`:

```python
from pydantic_settings import BaseSettings

class RAGConfig(BaseSettings):
    llm_model: str = "llama3.2"
    llm_base_url: str = "http://localhost:11434"
    # ...

    class Config:
        env_prefix = "RAG_"
```

Then set environment variables like `RAG_LLM_MODEL=llama3.1` to override defaults.

---

## 3. Error Handling and Structured Responses

We use FastAPI's `HTTPException` to provide consistent error handling across the API. Each endpoint catches potential failures and returns a structured error response with the appropriate HTTP status code.

Example error response:

```json
{
  "detail": "Image generation failed: Connection refused"
}
```

We can also add custom exception handlers for more granular control:

```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail}
    )
```

---

## 4. Testing the API

### With `curl`

**Health Check:**

```bash
curl http://localhost:8000/health
```

**Text Query:**

```bash
curl -X POST "http://localhost:8000/rag/query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the nutritional values of salmon?",
    "top_k": 5
  }'
```

**Multimodal Query:**

```bash
curl -X POST "http://localhost:8000/rag/multimodal" \
  -F "query=Explain this food and its nutritional values" \
  -F "image=@/path/to/food.jpg" \
  -F "top_k=5"
```

**Generate Image (Download):**

```bash
curl -X POST "http://localhost:8000/rag/generate-image" \
  -F "description=a plate of salmon with roasted vegetables" \
  --output salmon.png
```

**Generate Image (Save to Server):**

```bash
curl -X POST "http://localhost:8000/rag/generate-image" \
  -F "description=a plate of salmon with roasted vegetables" \
  -F "save_path=./generated/salmon.png"
```

**Streaming Query:**

```bash
curl --no-buffer -X POST "http://localhost:8000/rag/stream" \
  -H "Content-Type: application/json" \
  -d '{"query": "List the top 3 foods rich in vitamin C."}'
```

### With Postman

1. **Import the collection** : Use the JSON collection provided below.
2. **Set environment variables** (optional): e.g., `{{base_url}}` = `http://localhost:8000`.
3. **Run each request** individually.
4. The outputs from the multimodal query and image generation workflows are shown in [Figure 1](#fig1) and [Figure 2](#fig2), respectively.

**Postman Collection (JSON):**

```json
{
  "info": {
    "name": "Food RAG API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Health Check",
      "request": {
        "method": "GET",
        "url": "http://localhost:8000/health"
      }
    },
    {
      "name": "RAG Query",
      "request": {
        "method": "POST",
        "url": "http://localhost:8000/rag/query",
        "header": [{"key": "Content-Type", "value": "application/json"}],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"query\": \"What are the nutritional values of salmon?\",\n  \"top_k\": 5\n}"
        }
      }
    },
    {
      "name": "Multimodal Query",
      "request": {
        "method": "POST",
        "url": "http://localhost:8000/rag/multimodal",
        "body": {
          "mode": "formdata",
          "formdata": [
            {"key": "query", "value": "What food is this?", "type": "text"},
            {"key": "image", "src": "/path/to/test.jpg", "type": "file"}
          ]
        }
      }
    },
    {
      "name": "Generate Image",
      "request": {
        "method": "POST",
        "url": "http://localhost:8000/rag/generate-image",
        "body": {
          "mode": "formdata",
          "formdata": [
            {"key": "description", "value": "salade césar", "type": "text"}
          ]
        }
      }
    },
    {
      "name": "Stream Query",
      "request": {
        "method": "POST",
        "url": "http://localhost:8000/rag/stream",
        "header": [{"key": "Content-Type", "value": "application/json"}],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"query\": \"List the top 3 foods rich in vitamin C.\",\n  \"top_k\": 3\n}"
        }
      }
    }
  ]
}
```

<figure id="fig1">
  <img src="/images/posts/2026-06-21-part-3-exposing-the-rag-system-with-a-fastapi-backend/1-postman-query-rag-multimodal-fish.png" height="100%" weight="100%">
  <figcaption>Figure 1: Multimodal RAG Endpoint (POST).</figcaption>
</figure>

<figure id="fig2">
  <img src="/images/posts/2026-06-21-part-3-exposing-the-rag-system-with-a-fastapi-backend/2-postman-query-rag-generate-image-salade-césar.png" alt="query-rag-generate-image" height="100%" weight="100%">
  <figcaption>Figure 2: Generate Image RAG Endpoint (POST).</figcaption>
</figure>

---

## 5. Running the Server

### Using `uv` (Recommended)

```bash
# From the project root
uv run fastapi dev src/rag/rag_fastapi_app.py
```

Or with `uvicorn`:

```bash
uv run uvicorn src.rag.rag_fastapi_app:app --reload --port 8000
```

### Using `uv` from the `rag` directory

```bash
cd src/rag
uv run uvicorn rag_fastapi_app:app --reload --port 8000
```

### Using `python` directly

```bash
python -m uvicorn src.rag.rag_fastapi_app:app --reload --port 8000
```

### Environment Variables

Set configuration via environment variables (if using `BaseSettings`):

```bash
export RAG_LLM_MODEL=llama3.1
export RAG_DB_HOST=production-db.example.com
uv run uvicorn src.rag.rag_fastapi_app:app --host 0.0.0.0 --port 8000
```

---

## API Architecture Overview

The architecture is organized into three main layers [Figure 3](#fig3): the **client layer**, the **FastAPI application layer** , and the **RAG core layer**.

<figure id="fig3">
  <img src="/images/posts/2026-06-21-part-3-exposing-the-rag-system-with-a-fastapi-backend/3-rag-fastapi-architecture-flowchart.png" alt="rag-fastapi" height="100%" weight="100%">
  <figcaption>Figure 3: API Architecture.</figcaption>
</figure>

At the top level, a **Client** (such as a web application, mobile app, or API consumer) sends requests to the  **FastAPI Server** , which acts as the entry point to the system. The FastAPI application exposes several endpoints, each responsible for a specific capability:

* **`/health`** — Provides a health check endpoint to verify that the application is running and that required resources are available.
* **`/rag/query`** — Handles standard text-based RAG queries by receiving a user question and returning a generated answer.
* **`/rag/multimodal`** — Supports multimodal interactions by accepting both text and image inputs.
* **`/rag/generate-image`** — Generates food images from text descriptions using the image generation capability.
* **`/rag/stream`** — Provides streaming responses, sending LLM output progressively as tokens are generated.

All API endpoints communicate with the  **`RAGSystem`** , which serves as the central orchestrator of the application. Instead of implementing retrieval, prompt construction, and generation logic directly inside the API layer, the FastAPI server delegates these responsibilities to the RAG core.

The **RAG Core** consists of three main components:

1. **`ImageAwareFoodRetriever`**

* Responsible for retrieving relevant food information from the knowledge base.
* It communicates with  **PostgreSQL + pgvector** , which stores food metadata and vector embeddings generated in Part 1.
* The retriever performs semantic similarity searches to find foods relevant to the user's query.

2. **`FoodPromptBuilder`**

* Converts retrieved food documents into structured prompts.
* It extracts relevant nutritional information and formats the context that will be provided to the language model.

3. **`OllamaClient`**

* Acts as the interface between the RAG system and the local LLM service.
* It sends prompts to the  **Ollama Server** , which generates the final natural-language response.

This separation makes the system easier to maintain and extend. For example, the retrieval backend can be replaced without modifying the API layer, or the Ollama-based LLM client can be exchanged for another provider while keeping the rest of the pipeline unchanged.

---

## Advanced: Authentication and Rate Limiting (Optional)

For production deployments, you may want to add:

### API Key Authentication

```python
from fastapi import Security, HTTPAuthorizationCredentials
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

async def verify_api_key(api_key: str = Security(api_key_header)):
    if api_key != os.environ.get("API_KEY"):
        raise HTTPException(status_code=403, detail="Invalid API key")
    return api_key

@app.post("/rag/query", dependencies=[Depends(verify_api_key)])
async def rag_query(request: QueryRequest):
    # ...
```

### Rate Limiting

Using `slowapi`:

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(429, _rate_limit_exceeded_handler)

@app.post("/rag/query")
@limiter.limit("10/minute")
async def rag_query(request: QueryRequest, request: Request):
    # ...
```

---

## Conclusion

We've taken the modular RAG system from Part 2 and wrapped it in a production‑ready FastAPI backend. The API provides:

* **Text and multimodal queries** with structured JSON responses.
* **Image generation** with both download and server‑save modes.
* **Streaming responses** for real‑time user experiences.
* **Health checks** with database validation.
* **Lazy loading** for fast startup.
* **Temporary file cleanup** with `BackgroundTasks`.
* **Centralised configuration** for easy deployment.

The result is a deployable API that can be consumed by web frontends, mobile apps, or other microservices. The clean separation between the HTTP layer and the core RAG logic means we can evolve each independently.

### What's Next?

In **Part 4**, we'll develop an **Agentic RAG** architecture where an autonomous agent dynamically selects and adjusts its search strategy to retrieve the most relevant information.


### 📦 GitHub Repository

The complete code for this series, including the FastAPI backend, is available in the [Generative‑AI‑LLMs GitHub repository](https://github.com/tantikristanti/Generative-AI-LLMs/tree/main/food-nutrition-semantic-search-rag) under the `rag` package.

---

## References

1. ANSES. (2025).  **Ciqual French food composition table**. [https://ciqual.anses.fr/](https://ciqual.anses.fr/)
2. FastAPI. (2026).  **FastAPI framework**. [https://fastapi.tiangolo.com/](https://fastapi.tiangolo.com/)
3. Uvicorn. (2026).  **Uvicorn ASGI server**. [https://uvicorn.dev/](https://uvicorn.dev/)
4. Pydantic. (2026).  **Pydantic Settings management**. [https://docs.pydantic.dev/latest/concepts/pydantic_settings/](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
5. Ollama. (2026).  **Ollama API documentation**. [https://github.com/ollama/ollama/blob/main/docs/api.md](https://github.com/ollama/ollama/blob/main/docs/api.md)

---
