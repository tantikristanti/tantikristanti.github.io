---
title: 'Building a Production-Ready Food Nutrition Semantic Search and RAG System — Part 4: From Fixed Pipeline to Intelligent Multi-Agent RAG'
date: 2026-06-29
collection: posts
permalink: /posts/2026/06/part-4-from-fixed-pipeline-to-intelligent-multi-agent-rag/
toc: true
tags:
  - agentic-rag
  - multi-agent
  - modular-architecture
  - semantic-search
---
This is **Part 4** of the series **Building a Production-Ready Food Nutrition Semantic Search and RAG System**.

If you're new to the series, start with [Part 1: Persistent Semantic Search with FastAPI and pgvector](https://tantikristanti.github.io/posts/2026/06/part-1-persistent-semantic-search-with-fastapi-and-pgvector/), where we built the multilingual retrieval engine. In [Part 2: Designing a Modular Food RAG Architecture with Object‑Oriented Principles](https://tantikristanti.github.io/posts/2026/06/part-2-designing-a-modular-food-rag-architecture-with-object-oriented-principles/), we designed a modular RAG architecture using Object-Oriented Programming principles. In [Part 3: Exposing the RAG System with a FastAPI Backend](https://tantikristanti.github.io/posts/2026/06/part-3-exposing-the-rag-system-with-a-fastapi-backend/), we wrapped everything in a production-ready FastAPI backend.

By this point, we have a fully functional RAG pipeline: user queries are converted into embeddings, relevant food nutrition documents are retrieved from PostgreSQL using semantic search, and the retrieved context is passed to a local LLM running on Ollama to generate natural-language answers.

While this architecture works well for clear and well‑formed queries, it has an important limitation. Consider a user asking, **"What is the protien content of chicken breast?"** with *protein* misspelled, or submitting an ambiguous query that requires multiple search strategies. Our current system performs a single search using the user's exact input and proceeds with whatever results are returned.

This fixed retrieval pipeline lacks a mechanism for adapting when search quality is poor. If the query contains spelling mistakes, ambiguous wording, or retrieves irrelevant documents, the system has no way to reformulate the search, try alternative strategies, or recover from failure.

In this post, we'll address that limitation by introducing **Multi-Agent RAG**, an intelligent retrieval architecture in which specialised agents collaborate to evaluate search results, reformulate queries, and dynamically adapt their retrieval strategy to improve answer quality.

---

## Introduction: The Problem with Fixed-Pipeline RAG

Retrieval-Augmented Generation (RAG) has become the standard approach for grounding LLM responses in external knowledge. At its core, a traditional RAG pipeline follows three straightforward steps:

1. **Retrieve**: Search a knowledge base for relevant documents using vector similarity.
2. **Augment**: Inject the retrieved context into a structured prompt.
3. **Generate**: Pass the prompt to an LLM and return the generated answer.

For well‑formed queries, this workflow performs remarkably well. However, real‑world user input is rarely perfect — users misspell words, phrase questions in unexpected ways, omit important context, or ask ambiguous questions that require additional reasoning. Even with semantic search, these challenges can still lead to poor retrieval quality.

Common failure cases include:

* **Misspelled queries** (e.g., *"protien"* instead of *"protein"*)
* **Non‑standard wording** that differs significantly from the indexed content
* **Insufficient context**, where a single search is not enough to answer the question
* **Ambiguous requests** that require clarification or multiple retrieval attempts

To overcome these limitations, we can move beyond a fixed retrieval pipeline and introduce **Agentic RAG**. Instead of performing a single search and accepting the outcome, an agentic system can evaluate retrieval quality, reformulate queries, explore alternative search strategies, and iteratively recover from failures. The result is a more adaptive and resilient RAG architecture that behaves less like a static pipeline and more like an intelligent researcher.

---

## What Is Agentic RAG?

Instead of a fixed pipeline, an **agent** uses an iterative loop:

```
User Query → Agent decides → Call tool → Agent evaluates → Continue or Answer
```

The agent decides:

* **When** to call a tool
* **Which** tool to call
* **What arguments** to pass
* **When to stop** and provide an answer

Based on the [LLM Zoomcamp materials](https://github.com/DataTalksClub/llm-zoomcamp/blob/main/01-agentic-rag/lessons/14-agentic-loop.md), a well‑designed agent has three components:

1. **Instructions**: The developer message that defines the agent's role and behavior.
2. **Tools**: Functions the agent can call (e.g., search, correct_typo).
3. **Memory**: The message history that tracks what the agent has tried.

---

## Multi-Agent vs. Classic RAG: A Side‑by‑Side Comparison

Before exploring the architecture, it's important to understand how a multi‑agent system differs from a traditional RAG pipeline.

| Aspect | Classic RAG (Fixed Pipeline) | Multi-Agent RAG (This System) |
|--------|------------------------------|-------------------------------|
| **Retrieval strategy** | Single, fixed method (e.g., pure vector search) | Adaptive: can combine vector, keyword, and multi‑query searches |
| **Error recovery** | None – if retrieval fails, the answer is poor | Built‑in retry and reformulation (typo correction, query variations) |
| **Query understanding** | Passive – accepts the query as‑is | Active – refines, expands, and rephrases queries before searching |
| **Component interaction** | Linear and static | Dynamic message passing between specialised agents |
| **Best‑suited for** | Well‑formed, predictable queries | Noisy, ambiguous, or complex user inputs |
| **Latency / cost** | Lower (one LLM call + one search) | Higher (multiple tool calls and iterations) – a deliberate trade‑off for accuracy |
| **Failure recovery** | None – failures propagate to the answer | Graceful degradation – can fall back to alternative strategies |
| **Transparency** | Black box – hard to debug retrieval issues | Observable – all strategies used are logged and returned |

This comparison clarifies the key trade‑off: we accept higher latency and computational cost in exchange for significantly improved robustness and answer quality, especially when dealing with real‑world, imperfect queries.

---

## Multi‑Agent Architecture

Instead of relying on a single monolithic agent, we design a **multi‑agent architecture** with clearly defined responsibilities. The following diagram illustrates how the **Orchestrator Agent** manages the workflow by coordinating three specialised agents, each operating as an independent component at the same hierarchical level.

<figure id="fig1">
  <img src="/images/posts/2026-06-29-part-4-from-fixed-pipeline-to-intelligent-multi-agent-rag/1-agentic-rag-flowchart.png" alt="agentic-rag-flowchart" height="100%" weight="100%">
  <figcaption>Figure 1: Multi-Agent Architecture showing the Orchestrator Agent coordinating three specialised agents (Query Refiner, Search Agent, and Response Agent).</figcaption>
</figure>

---

### Agent 1: Search Agent

**Role:** Executes searches with adaptive strategies and built‑in query refinement.

**Key capabilities:**
- **Query refinement pipeline** (spelling correction, rewriting, variation generation)
- **Embedding‑based retrieval** on multiple query variations
- **Cross‑encoder reranking** of results
- **Raw search fallback** when refinement yields insufficient results

**Decision logic:**
1. Execute `refined_search()` — a comprehensive pipeline that handles spelling correction, query rewriting, variation generation, embedding search, and cross‑encoder reranking
2. If results < `min_results_threshold`, fall back to `raw_search()` as a safety net
3. Merge and re‑rank results from both strategies

**Example:** The query *"protien in chicken"* is corrected to *"protein in chicken"*, rewritten to *"What is the protein content of chicken?"*, and multiple variations are generated and searched.

---

### Agent 2: Response Agent

**Role:** Generates the final, well‑formatted answer from the retrieved documents.

**Tools:**
- `generate_answer(query, documents)` → Builds a context‑aware prompt and calls the LLM
- `suggest_follow_ups(query, documents)` → Proposes 2–3 related questions
- `format_response()` → Structures the final output with sources and citations

**Decision logic:**
- If no documents were retrieved → Respond with a polite "no results" message
- If documents are sufficient → Format them into the prompt, invoke the LLM, and append source citations
- Always include suggested follow‑up questions to improve user engagement

---

### Agent 3: Orchestrator Agent

The **Orchestrator Agent** is the main entry point that:

* Receives the user's raw query
* Calls the **Search Agent** to fetch context (with built‑in refinement)
* Passes the retrieved context to the **Response Agent** to draft the final answer
* Returns the final answer, complete with sources and follow‑ups, back to the user

The workflow follows this sequential data flow:
`Search Agent` → `Response Agent` → `Orchestrator`

---

## Multi-Agent RAG Class Diagram

[Figure 2](#fig2) presents the class diagram of the proposed Agentic RAG system. The design follows a modular object‑oriented architecture in which responsibilities are divided among agents, tools, retrieval components, and supporting data structures. This separation of concerns improves maintainability, extensibility, and code reuse.

<figure id="fig2">
  <img src="/images/posts/2026-06-29-part-4-from-fixed-pipeline-to-intelligent-multi-agent-rag/2-agentic-rag-class-diagram.png" alt="agentic-rag-class-diagram" height="100%" weight="100%">
  <figcaption>Figure 2: Multi-Agent RAG Class Diagram.</figcaption>
</figure>

At the highest level, the `AgenticRAGSystem` class represents the entry point of the application. It encapsulates the core components required for query processing, including the `SearchTools`, `ResponseTools`, and `OrchestratorAgent` objects. Additionally, it maintains references to the `ImageAwareFoodRetriever` and `OllamaClient`, which provide document retrieval and large language model (LLM) capabilities, respectively. The system exposes two public methods: ***query()***, which performs the complete retrieval and response generation workflow, and ***search_only()***, which retrieves relevant documents without generating a final answer.

The workflow is coordinated by the `OrchestratorAgent`, which inherits from the abstract `BaseAgent` class. Rather than performing retrieval or generation directly, the orchestrator manages the interaction between the search and response components. It receives the user's query, invokes the appropriate tools, and returns a structured `AgentResponse` object. The agent's behavior is governed by an `AgentConfig` object, which specifies operational parameters such as the maximum number of iterations, retrieval thresholds, and query refinement options.

The abstract `BaseAgent` class defines the common functionality shared by all agents. It manages conversation history through `AgentMessage` objects, registers available `Tool` instances, executes tool calls, and provides the generic interface ***process()*** that every concrete agent must implement. This abstraction allows additional specialized agents to be added without modifying the overall architecture.

The `SearchAgent`, another subclass of `BaseAgent`, is responsible for retrieving relevant documents. It relies on the `SearchTools` component to perform refined searches while enforcing configurable quality thresholds. The retrieved information is packaged into an `AgentResponse`, which records the retrieved documents, executed tool calls, and the search outcome.

The `SearchTools` class encapsulates all retrieval-related operations. It combines three major components: the `ImageAwareFoodRetriever`, the `OllamaClient`, and the `QueryTools` module. It provides methods for both raw document retrieval and refined retrieval, while maintaining a searchable history of previous searches. Search results are represented by the SearchResult class, which stores the query, retrieved documents, retrieval strategy, and any refined version of the original query.

Query preprocessing is handled by the `QueryTools` class. Before searching the knowledge base, it improves the user's query by detecting spelling mistakes, generating alternative query formulations, expanding queries with synonyms, and reranking retrieved documents. The output of these operations is represented by the `RefinedQuery` class, which preserves the original query together with any corrected, rewritten, or expanded versions used during retrieval.

Response generation is performed by the `ResponseTools` class. Using the `OllamaClient`, it synthesizes retrieved documents into a coherent natural-language answer. In addition to generating responses, it can suggest follow-up questions and format the final output for presentation. The resulting FormattedResponse object contains the generated answer, supporting source references, follow-up suggestions, and any refined query information used during processing.

The retrieval subsystem is built upon the abstract `BaseRetriever` class, which defines generic search operations for text and image queries. The concrete `ImageAwareFoodRetrieve` class implements these operations, enabling multimodal retrieval from the underlying knowledge base. Retrieved information is represented by the `RetrievedDocument` class, which stores the document content, metadata, similarity score, and optional image information.

Similarly, language model functionality is abstracted through the `BaseLLMClient` class. Its concrete implementation, `OllamaClient`, provides methods for text generation, multimodal reasoning, and image generation. This abstraction allows different LLM providers to be substituted without affecting the higher-level agent architecture.

Supporting classes facilitate communication between components. `AgentMessage` stores conversational exchanges and tool calls between agents and the language model, while `AgentResponse` records the final processing outcome, including generated content, execution history, iteration count, and error information when applicable. The `Tool` class represents callable functions that agents can invoke during reasoning, enabling flexible integration of external capabilities.

The relationships shown in the diagram reflect the modular design of the system. *Inheritance* is used to define reusable abstractions, with `SearchAgent` and `OrchestratorAgent` extending `BaseAgent`, `ImageAwareFoodRetriever` extending `BaseRetriever`, and `OllamaClient` extending `aseLLMClient`. *Composition* relationships indicate that the `AgenticRAGSystem` owns the principal components required for execution, including the orchestrator and tool modules. *Association* relationships represent dependencies between agents and their supporting tools, while *dependency* relationships indicate the creation and use of structured data objects such as `SearchResult`, `RefinedQuery`, `FormattedResponse`, and `AgentResponse`.

This design promotes extensibility by allowing individual modules to be modified or replaced without impacting the remainder of the system, making the Agentic RAG architecture easier to maintain and evolve than a tightly coupled monolithic implementation.

---

## Implementation Highlights

The `agentic_rag` package introduces a clean, modular implementation of the multi‑agent architecture. Below are the key components.

```text
src/agentic_rag/
├── __init__.py
├── config.py              # Configuration settings
├── tools/
│   ├── __init__.py
│   ├── search_tools.py    # Search-related tools
│   ├── query_tools.py     # Query refinement tools
│   └── response_tools.py  # Response generation tools
├── agents/
│   ├── __init__.py
│   ├── base_agent.py      # Abstract agent class
│   ├── query_refiner.py   # Query refinement agent
│   ├── search_agent.py    # Search execution agent
│   └── orchestrator.py    # Main orchestrator agent
├── memory/
│   ├── __init__.py
│   └── conversation_memory.py  # Message history management
└── agentic_rag_system.py  # Main entry point
```

---

### Configuration

The system is configured through a dataclass:

```python
# src/agentic_rag/config.py
@dataclass
class AgentConfig:
    """Configuration for agent behavior."""

    # Search settings
    max_search_attempts: int = 3
    min_results_threshold: int = 2
    default_top_k: int = 5

    # Agent loop settings
    max_iterations: int = 5

    # Model settings
    model_name: str = "llama3.2"
    embedding_model: str = "paraphrase-multilingual-MiniLM-L12-v2"

    # Query refinement
    enable_typo_detection: bool = True
    enable_query_expansion: bool = True
    max_query_variations: int = 3
```

---

### The Search Agent's Pipeline

The `SearchAgent` follows a two‑step strategy that balances sophistication with simplicity:

#### Step 1: Refined Search (Primary)

The agent first calls `refined_search()`, which is a **complete search pipeline** that handles:

* **Spelling correction** using [SymSpell](https://github.com/wolfgarbe/symspell)
* **Query rewriting** using the LLM
* **Query variation generation**
* **Embedding search** on all query variations
* **Cross‑encoder reranking** between the query and the document results

All of this intelligence is encapsulated in a single tool call, making the Search Agent's logic remarkably clean:

```python
# Primary: refined search
result = self.search_tools.refined_search(query)
strategies_used.append("refined")
docs = result.documents
```

#### Step 2: Raw Search Fallback

If the refined search returns **fewer than `min_results_threshold`** documents, the agent falls back to a raw vector search:

```python
if self.use_fallback:
    raw_result = self.search_tools.raw_search(query, top_k=self.min_results_threshold)
    strategies_used.append("raw_fallback")
    # Merge refined and raw docs, avoiding duplicates
```

The results from both searches are merged and re‑sorted by score, ensuring the user gets the best available context.

---

### Usage Example

```python
from src.agentic_rag import AgenticRAGSystem, AgentConfig
from src.rag.ollama_client import OllamaClient
from src.rag.retriever import ImageAwareFoodRetriever

# Initialize components
config = AgentConfig.default()
llm_client = OllamaClient(model="llama3.2")
retriever = ImageAwareFoodRetriever(
    model_name=config.embedding_model
)

# Create the agentic RAG system
system = AgenticRAGSystem(
    config=config,
    llm_client=llm_client,
    retriever=retriever
)

# Query with typo
result = system.query("What is the protein content of a mixed green salad with grilled fish?")

print(f"Answer: {result['answer']}")
print(f"Sources found: {result['total_sources']}")
print(f"Strategies used: {result['strategies_used']}")
# Example output:
# Strategies used: ['original', 'typo_correction', 'variation: chicken breast nutrition']
```

The response includes not only the answer but also metadata about *how* the answer was retrieved, making the system transparent and debuggable.

The following example demonstrates the application being executed and tested using [Postman](https://www.postman.com/). A JSON payload is sent in the body of a POST request, as shown in [Figure 3](#fig3):

```json
{
  "query": "What is the protien content of a mixed green salad with grilled fihs?",
  "top_k": 5,
  "model": "llama3.2",
  "temperature": 0.7
}
```

During testing, the `refined_search()` method successfully executed the complete query refinement pipeline ([Figure 3](#fig3)). The original user query, **"What is the protien content of a mixed green salad with grilled fihs?"**, contained two spelling errors ("protien" and "fihs"). The pipeline first applied **SymSpell** to automatically correct these errors, producing the corrected query:

*"What is the protein content of a mixed green salad with grilled fish?"*

Next, the **LLM-based query rewriting** stage reformulated the corrected query into a more natural and semantically expressive form:

*"What protein content can I expect in a mixed green salad topped with grilled fish?"*

The system then generated several **query variations** to broaden the retrieval space. These included alternative formulations, such as:

* *"What protein content can I expect in a mixed green salad topped with grilled fish?"*
* *"What nutritional benefits can I expect from eating a mixed green salad with grilled fish?"*

<figure id="fig3">
  <img src="/images/posts/2026-06-29-part-4-from-fixed-pipeline-to-intelligent-multi-agent-rag/3-postman-query-agentic-rag-refined-query.png" alt="postman-query-agentic-rag" height="100%" weight="100%">
  <figcaption>Figure 3: Multi-Agent RAG Query (POST) using Postman - Refined Search.</figcaption>
</figure>

These refined and expanded queries were subsequently used to perform **embedding-based retrieval**, enabling the system to identify documents that were semantically related to the user's intent rather than relying solely on exact keyword matching. Finally, the retrieved documents were evaluated using a **cross-encoder reranking model**, which reordered the results according to their semantic relevance before passing the highest-ranked documents to the response generation stage.

The example in [Figure 4](#fig4) demonstrates how the refinement pipeline improves retrieval quality by correcting typographical errors, reformulating ambiguous queries, and generating multiple semantic variations, thereby increasing the likelihood of retrieving relevant information from the knowledge base.

<figure id="fig4">
  <img src="/images/posts/2026-06-29-part-4-from-fixed-pipeline-to-intelligent-multi-agent-rag/4-postman-query-agentic-rag.png" alt="postman-query-agentic-rag" height="100%" weight="100%">
  <figcaption>Figure 4: Multi-Agent RAG Query (POST) using Postman - Query Responses.</figcaption>
</figure>

---

## When to Use Agents vs. Simple RAG

As noted in the [LLM Zoomcamp lessons](https://github.com/DataTalksClub/llm-zoomcamp/blob/main/01-agentic-rag/lessons/16-other-frameworks.md), agents come with costs:

* More API calls
* Higher latency
* More money spent
* Less predictable behavior

**Use the agentic approach when:**

* Users frequently make typos
* Queries are often ambiguous
* You need to handle diverse query patterns
* The cost of incorrect answers is high

**Stick with simple RAG when:**

* Queries are well-formed and consistent
* Speed is more important than accuracy
* You're operating under tight budget constraints
* Your domain is highly structured and predictable

---

## Next Steps & Improvements

1. **Add more tools** — Keyword search, hybrid search, image-based search
2. **Implement persistent memory** — Conversation memory across sessions
3. **Implement evaluation framework** — Track success rates and search strategies used
4. **Add streaming** — Stream responses as they're generated
5. **Integrate with LangChain** — Replace the vanilla implementation with LangChain for production
6. **A/B testing** — Compare agentic vs. classic RAG on real user queries
7. **Cost optimization** — Implement caching and result reuse to reduce API calls

---

## Conclusion

We've taken the modular RAG system from Part 2 and evolved it into an intelligent, adaptive multi-agent system. The agent can recover from typos, try multiple search strategies, combine results from different approaches, and provide transparent reasoning about its process—all while maintaining the clean, object-oriented design we established earlier.

The key insight is that agents are not a silver bullet, but a powerful tool for specific scenarios. They shine when queries are noisy, ambiguous, or complex. For straightforward, well-formed queries, the classic RAG pipeline remains perfectly adequate.

The complete code is available in the [Generative-AI-LLMs GitHub repository](https://github.com/tantikristanti/Generative-AI-LLMs/tree/main/food-nutrition-semantic-search-rag) under the `agentic_rag` package.

In the next installment, we'll explore how to deploy this agentic RAG system with proper monitoring, evaluation, and continuous improvement pipelines. We'll also discuss how to measure the real-world impact of agentic retrieval on user satisfaction and answer quality.

---

## References

1. ANSES. (2025).  **Ciqual French food composition table** . [https://ciqual.anses.fr/](https://ciqual.anses.fr/)
2. DataTalksClub. (2026).  **LLM Zoomcamp: Agentic RAG Module** . [https://github.com/DataTalksClub/llm-zoomcamp](https://github.com/DataTalksClub/llm-zoomcamp)
3. OpenAI. (2024).  **Function Calling Documentation** . [https://platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling)
4. Weng, L. (2023).  **LLM Powered Autonomous Agents** . [https://lilianweng.github.io/posts/2023-06-23-agent/](https://lilianweng.github.io/posts/2023-06-23-agent/)
5. LangChain. (2025).  **Agents Documentation** . [https://python.langchain.com/docs/modules/agents/](https://python.langchain.com/docs/modules/agents/)
6. Ollama. (2025).  **Ollama Documentation** . [https://ollama.com/](https://ollama.com/)
7. FastAPI. (2026).  **FastAPI Framework** . [https://fastapi.tiangolo.com/](https://fastapi.tiangolo.com/)

---

*This post is part of the series **Building a Production-Ready Food Nutrition Semantic Search and RAG System**.*

* *[Part 1: Persistent Semantic Search with FastAPI and pgvector](https://tantikristanti.github.io/posts/2026/06/part-1-persistent-semantic-search-with-fastapi-and-pgvector/)*
* *[Part 2: Designing a Modular Food RAG Architecture with Object-Oriented Principles](https://tantikristanti.github.io/posts/2026/06/part-2-designing-a-modular-food-rag-architecture-with-object-oriented-principles/)*
* *[Part 3: Exposing the RAG System with a FastAPI Backend](https://tantikristanti.github.io/posts/2026/06/part-3-exposing-the-rag-system-with-a-fastapi-backend/)*
* ***[Part 4: From Fixed Pipeline to Intelligent Multi-Agent RAG](https://tantikristanti.github.io/posts/2026/06/part-4-from-fixed-pipeline-to-intelligent-multi-agent-rag/)** (this post)*

---
