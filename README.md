# Scientific Literature RAG Pipeline

> **Commit Home Assignment** — A production-grade Retrieval-Augmented Generation (RAG) system that mines scientific papers from ArXiv and PubMed, builds a vector database, and answers complex research questions using LLM-powered query decomposition.

## Overview

This project demonstrates an end-to-end NLP pipeline built for a scientific corporation use case. Given a research topic, the system:

1. **Mines** scientific papers from ArXiv and PubMed
2. **Chunks** and **embeds** the extracted text into a vector database
3. **Answers complex questions** by decomposing them into sub-queries, retrieving relevant context, and synthesizing a final answer
4. **Evaluates** answer quality using a multi-dimensional RAGAS-inspired framework

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      DATA INGESTION                         │
│                                                             │
│  ArXiv / PubMed  ──▶  Web Scraper  ──▶  Text Extractor     │
│       URLs           (httpx async      (BeautifulSoup)      │
│                       + rate limit)                          │
│                              │                              │
│                              ▼                              │
│                       Text Chunker                          │
│                   (RecursiveCharTextSplitter)                │
│                              │                              │
│                              ▼                              │
│                      Embedding Model                        │
│                   (sentence-transformers)                    │
│                              │                              │
│                              ▼                              │
│                    ChromaDB Vector Store                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      QUERY PIPELINE                         │
│                                                             │
│  User Question  ──▶  Query Decomposer  ──▶  Sub-Queries     │
│                          (LLM)                              │
│                              │                              │
│                              ▼                              │
│                     Vector DB Retrieval                      │
│                      (per sub-query)                        │
│                              │                              │
│                              ▼                              │
│                     Answer Synthesizer                       │
│                      (LLM + citations)                      │
│                              │                              │
│                              ▼                              │
│                       Final Answer                          │
└─────────────────────────────────────────────────────────────┘
```

## Tasks Breakdown

### Task 1 — Web Mining & Vector Database

- Scrapes scientific papers from **ArXiv** (abstracts + full HTML) and **PubMed** (abstracts) using async `httpx` with semaphore-based rate limiting
- Extracts and cleans text with **BeautifulSoup**
- Splits text into chunks using `RecursiveCharacterTextSplitter` (500 chars, 100 overlap)
- Embeds chunks with `sentence-transformers` (`all-MiniLM-L6-v2`)
- Stores embeddings in **ChromaDB** with cosine similarity

### Task 2 — Complex Question Answering

Implements a **multi-step RAG pipeline** with query decomposition:

1. **Decompose** — An LLM breaks the complex question into 2–5 atomic sub-questions
2. **Retrieve** — Each sub-question queries the vector DB independently; results are deduplicated
3. **Synthesize** — An LLM composes a final answer from all retrieved context, grounded in the source material with citations

### Task 3 — Production Hardening

Refactors the pipeline into a `ProductionRAGPipeline` class with:

| Category           | Implementation                                                   |
| ------------------ | ---------------------------------------------------------------- |
| Logging            | Structured logging with levels, timestamps, component names      |
| Error Handling     | Try/catch at every stage with graceful degradation               |
| Retries            | `httpx` transport-level retries                                  |
| Connection Pooling | `httpx.Limits` for max connections and keepalive management      |
| Configuration      | Centralized `AppConfig` dataclass (no hardcoded values)          |
| Input Validation   | Query length limits, URL allowlisting, control char sanitization |
| Idempotency        | `upsert` instead of `add` — safe to re-run ingestion             |
| Metrics            | Tracks success/failures, latency, token usage, errors            |
| Health Checks      | `health_check()` endpoint for k8s / load balancer probes         |
| Persistent Storage | ChromaDB with persistent directory (survives restarts)           |

### Task 4 — RAG Evaluation

Implements a `RAGEvaluator` class with four metrics (RAGAS-inspired):

| Metric                | What It Measures                                            | Method       |
| --------------------- | ----------------------------------------------------------- | ------------ |
| Retrieval Precision@K | Are retrieved docs relevant to the query?                   | LLM-as-Judge |
| Faithfulness          | Is the answer grounded in context? (catches hallucinations) | LLM-as-Judge |
| Answer Relevance      | Does the answer address the original question?              | LLM-as-Judge |
| Answer Correctness    | Is the answer factually correct vs. gold standard?          | LLM-as-Judge |

A **composite score** aggregates all metrics for an overall quality assessment.

## Tech Stack

| Component       | Technology                                   |
| --------------- | -------------------------------------------- |
| Web Scraping    | `httpx` (async), `BeautifulSoup`             |
| Text Splitting  | `langchain-text-splitters`                   |
| Embeddings      | `sentence-transformers` (`all-MiniLM-L6-v2`) |
| Vector Database | `ChromaDB`                                   |
| LLM             | OpenAI `gpt-3.5-turbo`                       |
| Orchestration   | Python `asyncio`                             |

## Setup

### Prerequisites

- Python 3.9+
- An OpenAI API key

### Installation

```bash
pip install httpx beautifulsoup4 chromadb sentence-transformers langchain langchain-text-splitters openai tiktoken
```

### Configuration

Set your OpenAI API key as an environment variable:

```bash
export OPENAI_API_KEY="your-api-key-here"
```

### Running

Open the notebook in **Google Colab** or **Jupyter** and run the cells sequentially. The notebook is self-contained — all tasks build on each other progressively.

## Example Query

```
Question: "What are the dangerous medical conditions that AI systems might
           misdiagnose, and what safety measures should be put in place?"

Pipeline:
  1. Decomposes into sub-questions (e.g., "What medical conditions are
     commonly misdiagnosed?", "What safety measures exist for clinical AI?")
  2. Retrieves relevant chunks from the vector DB for each sub-question
  3. Synthesizes a cited, comprehensive answer
```

## License

This project was created as a take-home assignment for [Commit](https://www.commit-ai.io/).
