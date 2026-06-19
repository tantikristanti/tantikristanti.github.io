---
title: 'Building a Production-Ready Food Nutrition Semantic Search and RAG System — Part 2: Designing a Modular Food RAG Architecture with Object-Oriented Principles'
date: 2026-06-19
collection: posts
permalink: /posts/2026/06/part-2-designing-a-modular-food-rag-architecture-with-object-oriented-principles/
toc: true
tags:
  - rag
  - llm
  - modular-architecture
  - semantic-search
---
This is **Part 2** of the series ***Building a Production-Ready Food Nutrition Semantic Search and RAG System***. If you're new to the series, start with [Part 1: Persistent Semantic Search with FastAPI and pgvector](https://tantikristanti.github.io/posts/2026/06/part-1-persistent-semantic-search-with-fastapi-and-pgvector/), where we built the multilingual retrieval engine that powers the system.

In [Part 1](https://tantikristanti.github.io/posts/2026/06/part-1-persistent-semantic-search-with-fastapi-and-pgvector/), we built a persistent semantic search engine that transformed the [French CIQUAL food composition dataset](https://ciqual.anses.fr/)[[1][1]] into a RAG‑ready retrieval API. We ended with a FastAPI service that could answer natural language queries like *"What foods are rich in omega‑3?"* by finding semantically similar foods using pgvector.

However, retrieval is only half the story. A complete RAG (Retrieval-Augmented Generation) system needs to take those retrieved documents, feed them to a large language model (LLM), and generate a coherent natural language answer. This post picks up exactly where the search engine left off. We build a full RAG system that:

* Retrieves relevant foods from PostgreSQL using semantic search.
* Formats the retrieved data into a structured prompt.
* Sends the prompt to a local LLM (Ollama) and returns the answer.

More importantly, we design the entire system using **Object-Oriented Programming (OOP)** principles. The result is a codebase that is maintainable, testable, and extendable, ready to adapt as our needs evolve and as new LLM
providers emerge.

In a [previous post](https://tantikristanti.github.io/posts/2026/06/persistent-semantic-search-building-a-rag-ready-food-engine-with-fastapi-and-pgvector/), we built a persistent semantic search engine that transformed the [French CIQUAL food composition dataset](https://ciqual.anses.fr/) into a RAG-ready retrieval API. We ended with a FastAPI service that could answer natural language queries like *"What foods are rich in omega‑3?"* by finding semantically similar foods using pgvector.

However, retrieval is only half the story. A complete RAG (Retrieval-Augmented Generation) system needs to take those retrieved documents, feed them to a large language model (LLM), and generate a coherent natural language answer. This post picks up where the search engine left off. We build a full RAG system that:

* Retrieves relevant foods from PostgreSQL using semantic search.
* Formats the retrieved data into a structured prompt.
* Sends the prompt to a local LLM (Ollama) and returns the answer.

More importantly, we design the entire system using **Object-Oriented Programming (OOP)** principles. The result is a codebase that is maintainable, testable, and extendable, ready to adapt as our needs evolve and as new LLM providers emerge.

---

## Introduction

RAG systems have become the de facto standard for grounding LLM responses in factual, domain-specific data. Instead of relying solely on the model's parametric memory, a RAG system retrieves relevant documents from an external knowledge base and includes them in the prompt. This approach reduces hallucinations, improves accuracy, and allows the system to
reference up‑to‑date information.

Many RAG tutorials implement everything in a single monolithic script: connect to the database, generate embeddings, call the LLM, and parse the response, all in one file. This works for demos, but it falls apart when we need to:

* **Swap the vector database** (e.g., from PostgreSQL to Pinecone).
* **Change the LLM** (e.g., from Ollama to OpenAI or Anthropic).
* **Modify the prompt template** without touching the retrieval logic.
* **Write unit tests** for individual components.

The solution to these problems is **Object‑Oriented Programming**. By designing our system around abstract contracts (interfaces) and concrete implementations, we decouple the components. The orchestrator doesn't care *how* the retriever finds documents or *how* the LLM generates text, it only cares that they implement the expected methods.

This post walks through the complete RAG implementation, explaining each component through the lens of OOP concepts:  **Abstraction**, **Inheritance**,  **Polymorphism**, **Composition**, and **Encapsulation**. The complete code is available in the [Generative‑AI‑LLMs GitHub repository](https://github.com/tantikristanti/Generative-AI-LLMs/tree/main/food-nutrition-semantic-search-rag) under the `rag` package.

---

## Why OOP for RAG? (A Comparison)

Before diving into the code, let's consider why OOP matters for RAG systems. The following table compares a monolithic approach with our modular OOP design:

| Aspect                        | Monolithic Script                           | OOP Modular Design                                 |
| ----------------------------- | ------------------------------------------- | -------------------------------------------------- |
| **Retriever swap**      | Rewrite database queries throughout         | Implement a new class inheriting `BaseRetriever` |
| **LLM provider change** | Modify HTTP calls in multiple places        | Swap the `llm_client` instance                   |
| **Prompt changes**      | Edit string formatting inside the main loop | Modify only the `PromptBuilder` class            |
| **Testing**             | Requires full system setup                  | Mock individual components                         |
| **Code readability**    | Hard to follow                              | Clear separation of concerns                       |
| **Extensibility**       | High risk of breaking existing features     | Add new components without touching others         |

---

## The Blueprints: Abstract Base Classes (`rag_base.py`)

> **OOP Concept: Abstraction & Inheritance**

The foundation of our system is a set of abstract base classes (ABCs). These files don't do any real work. Instead, they define  *contracts* : "If you want to be a Retriever, you must have a `search()` method. If you want to be an LLM Client, you must have a `generate()` method."

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Optional, Dict, Any

@dataclass
class RetrievedDocument:
    """Standard container for retrieved documents."""
    content: str
    metadata: Dict[str, Any]
    score: Optional[float] = None

@dataclass
class RAGResponse:
    """Standard container for RAG system output."""
    answer: str
    documents: List[RetrievedDocument]
    query: str

class BaseRetriever(ABC):
    """Abstract contract for all retrievers."""
    @abstractmethod
    def search(self, query: str, top_k: int = 5) -> List[RetrievedDocument]:
        pass

class BasePromptBuilder(ABC):
    """Abstract contract for all prompt builders."""
    @abstractmethod
    def build_user_prompt(self, query: str, documents: List[RetrievedDocument]) -> str:
        pass

class BaseLLMClient(ABC):
    """Abstract contract for all LLM clients."""
    @abstractmethod
    def generate(self, prompt: str, system_prompt: Optional[str] = None) -> str:
        pass
```

The abstraction allows the orchestrator to depend on a well-defined interface rather than a specific implementation. It doesn't need to know how a retriever performs its search; it only needs to know that a `search()` method is available. As a result, we can replace the PostgreSQL retriever with a different implementation in the future without making any changes to the orchestrator, provided it adheres to the same interface.

The `RetrievedDocument` and `RAGResponse` dataclasses standardise how information is passed between classes. This is a form of **encapsulation** where the data structure is defined in one place and reused everywhere.

---

## The Workers: Concrete Implementations

### 1. The Database Expert (`food_image_retriever.py`)

> **OOP Concept: Polymorphism & Dependency Injection**

This class (`ImageAwareFoodRetriever`) inherits from `BaseRetriever`. It is the only component that talks to our PostgreSQL database.

```python
class ImageAwareFoodRetriever(BaseRetriever):
    def __init__(self, model_name: str = "paraphrase-multilingual-MiniLM-L12-v2"):
        self.model = SentenceTransformer(model_name)
        self._connect_db()
        self.image_handler = self.ImageHandler()

    def search(self, query: str, top_k: int = 5) -> List[RetrievedDocument]:
        # 1. Encode query to vector
        query_emb = self.model.encode([query])[0]
        query_vec = '[' + ','.join(str(x) for x in query_emb) + ']'

        # 2. SQL query with cosine similarity
        sql = """
            SELECT alim_code, alim_nom_fr, alim_nom_eng, 
                   composition_text, metadata,
                   1 - (embedding <=> %s::vector) AS similarity
            FROM food_composition_embeddings
            ORDER BY embedding <=> %s::vector LIMIT %s;
        """
        self.cur.execute(sql, (query_vec, query_vec, top_k))
        rows = self.cur.fetchall()

        # 3. Package results as RetrievedDocument objects
        results = []
        for row in rows:
            metadata = row[4]  # JSONB field
            # Fetch image if available
            image_url = self.image_handler.get_image(row[0]) # alim_code
            if image_url:
                metadata['image_url'] = image_url

            results.append(RetrievedDocument(
                content=row[3],  # composition_text
                metadata=metadata,
                score=row[5]     # similarity
            ))
        return results

    class ImageHandler:
        """Inner class for image fetching — encapsulation in action."""
        def get_image_url(self, alim_code: int) -> Optional[str]:
            # Check local cache, then fallback to Open Food Facts
            ...
```

The `ImageHandler` inner class demonstrates **encapsulation**: the internal logic of fetching images (cache lookup, URL construction, error handling) is hidden inside the class. The rest of the system doesn't care *how* images are fetched. It just calls `get_image_url()` and gets a URL or `None`.

### 2. The Prompt Engineer (`prompt_builder.py`)

> **OOP Concept: Separation of Concerns**

This class takes the raw data (the user query and the list of retrieved documents) and formats it into a natural language prompt that the LLM can understand.

```python
class FoodPromptBuilder(BasePromptBuilder):
    def build_user_prompt(self, query: str, documents: List[RetrievedDocument]) -> str:
        if not documents:
            return f"User question: {query}\n\nNo relevant food data found."

        context_parts = []
        for i, doc in enumerate(documents, 1):
            metadata = doc.metadata
            # Extract key nutrient values from metadata
            row_data = metadata.get('row_data', {})
            nutrients = self._extract_nutrients(row_data)

            food_text = (
                f"{i}. Food: {metadata.get('alim_nom_eng', 'Unknown')} "
                f"(French: {metadata.get('alim_nom_fr', 'Unknown')})\n"
                f"   Energy: {nutrients.get('energy_kj', 'N/A')} kJ, "
                f"Protein: {nutrients.get('protein', 'N/A')} g, "
                f"Fat: {nutrients.get('fat', 'N/A')} g, "
                f"Carbohydrates: {nutrients.get('carbs', 'N/A')} g\n"
                f"Similarity score: {doc.score:.3f}"
            )
            context_parts.append(food_text)

        context = "\n\n".join(context_parts)
        return f"""Retrieved foods from the CIQUAL database:
        {context}

        Based on the food data above, please answer the following question:
        {query}

        Provide a clear, concise answer referencing the specific foods from the retrieved data."""
```

This class isolates the *prompt engineering* logic. If we want to change how the context is presented, for example from paragraphs to a table, or from English to French, we only change this file. The rest of the system remains untouched.

### 3. The Communicator (`ollama_client.py`)

> **OOP Concept: Encapsulation**

This class handles communication with the locally running Ollama server. It encapsulates the HTTP request logic.

```python
class OllamaClient(BaseLLMClient):
    def __init__(self, model: str = "llama3.2", base_url: str = "http://localhost:11434"):
        self.model = model
        self.base_url = base_url

    def generate(self, prompt: str, system_prompt: Optional[str] = None) -> str:
        payload = {
            "model": self.model,
            "prompt": prompt,
            "stream": False
        }
        response = requests.post(f"{self.base_url}/api/generate", json=payload)
        response.raise_for_status()
        return response.json().get("response", "")

    def generate_multimodal(self, prompt: str, images: List[str]) -> str:
        """Special method for vision models like LLaVA."""
        # Encode images to base64 and send alongside text
        ...
```

The rest of our system doesn't know about HTTP headers, JSON formatting, or error handling. It just calls `generate()` and gets a string back. This is **encapsulation** at its finest as the complexity is hidden behind a simple interface.

---

## The Orchestrator (`rag_system.py`)

> **OOP Concept: Composition**

This is the main class to interact with (`RAGSystem`). It doesn't do the heavy lifting itself; instead, it *composes* the workers together.

```python
class RAGSystem:
    def __init__(
        self,
        retriever: Optional[BaseRetriever] = None,
        prompt_builder: Optional[BasePromptBuilder] = None,
        llm_client: Optional[BaseLLMClient] = None,
        config: Optional[RAGConfig] = None
    ):
        self.config = config or RAGConfig()
        # Dependency Injection: if not provided, create defaults
        self.retriever = retriever or ImageAwareFoodRetriever()
        self.prompt_builder = prompt_builder or FoodPromptBuilder()
        self.llm_client = llm_client or OllamaClient(model=self.config.llm_model)

    def query(self, question: str, top_k: int = 5) -> RAGResponse:
        """Main pipeline: retrieve → build prompt → generate."""
        # 1. Retrieve relevant documents
        documents = self.retriever.search(question, top_k=top_k)

        # 2. Build the prompt
        user_prompt = self.prompt_builder.build_user_prompt(question, documents)

        # 3. Generate the answer
        answer = self.llm_client.generate(user_prompt, system_prompt=self.config.system_prompt)

        # 4. Package and return
        return RAGResponse(
            answer=answer,
            documents=documents,
            query=question
        )
```

The orchestrator depends on interfaces rather than concrete implementations. It doesn't matter whether the retriever uses PostgreSQL, FAISS, or another retrieval backend, as long as it provides a `search()` method. Similarly, it doesn't matter whether the llm_client communicates with Ollama, OpenAI, or another provider, as long as it implements `generate()`.

This design follows the principle of **dependency injection**, where the orchestrator receives its dependencies during initialization instead of creating them itself. One of the main benefits is improved testability. For example, we can inject a mock retriever that returns predefined documents, allowing us to test the prompt builder and LLM client in isolation without relying on a database or external services.

---

## The Package Entry Point

`__init__.py` file defines the package's public interface. When someone writes `from rag import RAGSystem`, `__init__.py` determines which objects are exposed to the user. It also uses lazy loading to import components only when needed, reducing startup overhead and memory usage.

---

## The Central Configuration Hub

The `RAGConfig` dataclass serves as a single source of truth for application settings, including model configuration, generation parameters, retrieval limits, and database names. Centralizing these values makes the system easier to maintain and eliminates the need to search through multiple files when updating configuration options.

---

## The Complete Execution Flow (Step-by-Step)

To understand how the system works, let's follow a user query from submission to response generation:

1. **User Submits a Query**
   The process begins when a user executes:

```python
rag.query("What are the nutritional values of salmon?")
```

2. **The Orchestrator Takes Control**
   The `RAGSystem` receives the query and coordinates the workflow, delegating tasks to the appropriate components.
3. **The Retriever Searches the Knowledge Base**
   The `ImageAwareFoodRetriever` processes the query by:

   * Converting the term *"salmon"* into a 384-dimensional embedding vector.
   * Performing a semantic similarity search using PostgreSQL and the `<=>` cosine distance operator.
   * Retrieving the most relevant records from the database.
   * Transforming the results into structured `RetrievedDocument` objects.
4. **The Prompt Builder Creates Context**
   The `FoodPromptBuilder` extracts relevant nutritional information, such as protein and fat content, from the retrieved documents and assembles it into a structured prompt for the language model.
5. **The LLM Generates an Answer**
   The `OllamaClient` sends the prompt to the running Ollama model, waits for the generated response, and returns the result to the orchestrator.
6. **The Response Is Assembled**
   The `RAGSystem` combines the generated answer with the supporting retrieved documents and packages everything into a `RAGResponse` object.
7. **The User Receives the Result**
   Finally, the completed response is returned to the user, along with the contextual information used to generate it.

---

## Conclusion

In this second installment of the series, we extended the persistent semantic search engine from [Part 1: Persistent Semantic Search with FastAPI and pgvector](https://tantikristanti.github.io/posts/2026/06/part-1-persistent-semantic-search-with-fastapi-and-pgvector/) into a complete RAG system built on a modular Object-Oriented architecture.

By designing the system around abstract interfaces and clear component boundaries, we created a pipeline where individual components can be replaced, tested, and extended independently. The orchestrator focuses only on coordinating the workflow, while each component is responsible for its own specific role.

The key design principles behind this architecture are:

* **Abstraction** — Abstract base classes define clear contracts between components, allowing the system to depend on interfaces rather than concrete implementations.
* **Inheritance** — Specialized implementations, such as an image-aware food retriever, can extend the base components while remaining compatible with the overall architecture.
* **Polymorphism** — Different implementations, such as LLM providers or retrieval backends, can be exchanged without requiring changes to the orchestrator.
* **Composition** — The orchestrator assembles independent components into a complete RAG pipeline while keeping the business logic clean and maintainable.
* **Encapsulation** — Complex implementation details are hidden behind simple interfaces, making each component easier to understand and modify.

At this stage, we have moved from a semantic search engine to a modular RAG foundation that is ready to be exposed through application interfaces and extended with additional capabilities.

In **Part 3**, we'll build a production-ready FastAPI backend around this RAG system, adding features such as health checks, file uploads, and streaming responses. Then, in **Part 4**, we'll create an interactive user interface with Gradio, allowing non-technical users to query the system through a simple conversational interface.


### 📦 GitHub Repository

The complete, runnable code for the RAG system, including the retriever, prompt builder, Ollama client, orchestrator, and the FastAPI server we'll build next, is available in the [Generative‑AI‑LLMs GitHub repository](https://github.com/tantikristanti/Generative-AI-LLMs/tree/main/food-nutrition-semantic-search-rag) under the `rag` package.

---

## References

1. ANSES. (2025). **Ciqual French food composition table**. [https://ciqual.anses.fr/](https://ciqual.anses.fr/).

---

[1]: https://ciqual.anses.fr/#/cms/download/node/20
