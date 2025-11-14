# Phase 3 Implementation Plan: Full Automation (Months 7-9)

> **Goal:** Automated research and RAG capabilities for end-to-end workflows
> **Timeline:** 12 weeks (3 months)
> **Prerequisites:** Phase 2 complete (production-ready system with web UI)
> **Deliverable:** Fully automated pipeline from business task to deployed feature

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Month 7: RAG Engine](#month-7-rag-engine)
4. [Month 8: Research Automation](#month-8-research-automation)
5. [Month 9: End-to-End Workflows](#month-9-end-to-end-workflows)
6. [RAG Best Practices](#rag-best-practices)
7. [Performance Optimization](#performance-optimization)

---

## Overview

### What We're Building in Phase 3

```
┌─────────────────────────────────────────────────────────────┐
│                     PHASE 3 SCOPE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Business Task (e.g., "Analyze grant opportunities")        │
│         ↓                                                   │
│  NEW: Research-Harvester-MCP (Web Search, Scraping)         │
│         ↓                                                   │
│  NEW: RAG-Engine-MCP (Semantic Search, Context)             │
│         ↓                                                   │
│  Spec-Synthesizer-MCP (with RAG context)                    │
│         ↓                                                   │
│  Codesmith-MCP (with codebase context from RAG)             │
│         ↓                                                   │
│  Sandbox-Runner-MCP → DevOps-Conductor-MCP                  │
│                                                             │
│  RESULT: Fully automated development from idea to deploy    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### New Services

1. **RAG-Engine-MCP** - Vector database, embeddings, semantic search
2. **Research-Harvester-MCP** - Automated web research and content aggregation

### Enhanced Services

1. **Spec-Synthesizer-MCP** - Use RAG for better context
2. **Codesmith-MCP** - Use RAG to understand existing codebase
3. **Vibes-Director** - Add end-to-end workflow templates

---

## Prerequisites

### Phase 2 Completion Checklist

- ✅ Production deployment complete
- ✅ Control-Hub-Web operational
- ✅ RBAC and cost quotas working
- ✅ DevOps-Conductor-MCP deployed
- ✅ Team onboarded and using system

### New Requirements for Phase 3

**Infrastructure:**
- Additional 16GB RAM (for vector database)
- 100GB+ storage (for embeddings and documents)
- GPU (optional, for faster embedding generation)

**Software:**
- Weaviate or Chroma (vector database)
- SearXNG (self-hosted search engine)

**Accounts:**
- OpenAI API key (for embeddings) OR use local models
- Optional: Perplexity API (if budget allows)

---

## Month 7: RAG Engine

### Week 1-2: Vector Database Setup

#### Task 7.1: Deploy Weaviate

**File:** `infrastructure/docker/docker-compose.yml` (add to services)

```yaml
  # Weaviate - Vector Database
  weaviate:
    image: semitechnologies/weaviate:1.23.0
    container_name: ecosystem-weaviate
    environment:
      QUERY_DEFAULTS_LIMIT: 25
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'false'
      AUTHENTICATION_APIKEY_ENABLED: 'true'
      AUTHENTICATION_APIKEY_ALLOWED_KEYS: '${WEAVIATE_API_KEY:-dev-key-change-me}'
      AUTHENTICATION_APIKEY_USERS: 'admin'
      PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
      DEFAULT_VECTORIZER_MODULE: 'none'  # We'll provide vectors
      ENABLE_MODULES: 'backup-filesystem,text2vec-openai'
      CLUSTER_HOSTNAME: 'node1'
    ports:
      - "8081:8080"
    volumes:
      - weaviate_data:/var/lib/weaviate
    networks:
      - ecosystem-network

volumes:
  weaviate_data:
```

#### Task 7.2: RAG-Engine-MCP Implementation

**File:** `services/rag-engine-mcp/requirements.txt`

```
fastapi==0.109.0
uvicorn==0.27.0
pydantic==2.6.0
weaviate-client==4.4.0
sentence-transformers==2.3.1
openai==1.10.0
tiktoken==0.5.2
numpy==1.26.3
```

**File:** `services/rag-engine-mcp/main.py`

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List, Dict, Any, Optional, Literal
import weaviate
from sentence_transformers import SentenceTransformer
import numpy as np
import os
import logging
from datetime import datetime

logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO").upper())
logger = logging.getLogger(__name__)

app = FastAPI(title="RAG Engine MCP")

# Configuration
WEAVIATE_URL = os.getenv("WEAVIATE_URL", "http://weaviate:8080")
WEAVIATE_API_KEY = os.getenv("WEAVIATE_API_KEY")
EMBEDDING_MODEL = os.getenv("EMBEDDING_MODEL", "BAAI/bge-large-en-v1.5")
USE_GPU = os.getenv("USE_GPU", "false").lower() == "true"

# Initialize embedding model (local)
logger.info(f"Loading embedding model: {EMBEDDING_MODEL}")
embedding_model = SentenceTransformer(
    EMBEDDING_MODEL,
    device="cuda" if USE_GPU else "cpu"
)
EMBEDDING_DIM = embedding_model.get_sentence_embedding_dimension()
logger.info(f"Embedding dimension: {EMBEDDING_DIM}")

# Initialize Weaviate client
weaviate_client = weaviate.Client(
    url=WEAVIATE_URL,
    auth_client_secret=weaviate.AuthApiKey(api_key=WEAVIATE_API_KEY) if WEAVIATE_API_KEY else None,
)


# Models
class Document(BaseModel):
    id: Optional[str] = None
    content: str
    metadata: Dict[str, Any] = {}
    namespace: str = "default"


class IngestDocumentsRequest(BaseModel):
    documents: List[Document]
    namespace: str = "default"
    chunk_size: int = 512
    chunk_overlap: int = 50


class SearchRequest(BaseModel):
    query: str
    top_k: int = Field(5, ge=1, le=50)
    namespace: str = "default"
    filters: Optional[Dict[str, Any]] = None
    min_score: float = Field(0.0, ge=0.0, le=1.0)


class SearchResult(BaseModel):
    id: str
    content: str
    score: float
    metadata: Dict[str, Any]


class HybridSearchRequest(SearchRequest):
    alpha: float = Field(0.5, ge=0.0, le=1.0)  # 0 = pure keyword, 1 = pure semantic


class AgenticRetrievalRequest(BaseModel):
    query: str
    max_iterations: int = Field(3, ge=1, le=10)
    namespace: str = "default"


class RAGRequest(BaseModel):
    query: str
    top_k: int = 5
    namespace: str = "default"
    system_prompt: Optional[str] = None


# Schema initialization
SCHEMA = {
    "class": "Document",
    "description": "A document with vector embedding",
    "vectorizer": "none",  # We provide vectors
    "properties": [
        {
            "name": "content",
            "dataType": ["text"],
            "description": "The document content",
        },
        {
            "name": "metadata",
            "dataType": ["object"],
            "description": "Document metadata",
        },
        {
            "name": "namespace",
            "dataType": ["string"],
            "description": "Namespace for multi-tenancy",
        },
        {
            "name": "created_at",
            "dataType": ["date"],
            "description": "Creation timestamp",
        },
    ],
}


def init_schema():
    """Initialize Weaviate schema."""
    try:
        # Check if class exists
        schema = weaviate_client.schema.get()
        classes = [c["class"] for c in schema.get("classes", [])]

        if "Document" not in classes:
            logger.info("Creating Document class in Weaviate")
            weaviate_client.schema.create_class(SCHEMA)
        else:
            logger.info("Document class already exists")
    except Exception as e:
        logger.error(f"Error initializing schema: {e}")


# Initialize on startup
@app.on_event("startup")
async def startup_event():
    init_schema()


@app.get("/")
async def root():
    return {
        "name": "rag-engine-mcp",
        "version": "0.1.0",
        "protocol": "mcp",
        "embedding_model": EMBEDDING_MODEL,
        "embedding_dim": EMBEDDING_DIM,
        "tools": [
            {
                "name": "ingest_documents",
                "description": "Ingest and embed documents into vector database",
            },
            {
                "name": "search_similar",
                "description": "Search for similar documents using semantic search",
            },
            {
                "name": "hybrid_search",
                "description": "Search using both semantic and keyword matching",
            },
            {
                "name": "agentic_retrieval",
                "description": "Multi-step retrieval with reasoning",
            },
            {
                "name": "retrieve_and_generate",
                "description": "RAG: Retrieve context and generate response",
            },
        ],
    }


@app.get("/health")
async def health():
    # Check Weaviate connection
    try:
        weaviate_client.schema.get()
        weaviate_status = "healthy"
    except Exception as e:
        weaviate_status = f"unhealthy: {str(e)}"

    return {
        "status": "healthy" if weaviate_status == "healthy" else "degraded",
        "weaviate": weaviate_status,
        "timestamp": datetime.utcnow().isoformat(),
    }


def chunk_text(text: str, chunk_size: int = 512, overlap: int = 50) -> List[str]:
    """Chunk text into smaller pieces with overlap."""
    words = text.split()
    chunks = []

    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        if chunk:
            chunks.append(chunk)

    return chunks


@app.post("/tools/ingest_documents")
async def ingest_documents(request: IngestDocumentsRequest):
    """Ingest documents into vector database with embeddings."""
    logger.info(f"Ingesting {len(request.documents)} documents into namespace '{request.namespace}'")

    ingested_count = 0
    chunk_count = 0

    try:
        with weaviate_client.batch as batch:
            batch.batch_size = 100

            for doc in request.documents:
                # Chunk document if it's long
                chunks = chunk_text(doc.content, request.chunk_size, request.chunk_overlap)

                for i, chunk in enumerate(chunks):
                    # Generate embedding
                    embedding = embedding_model.encode(chunk, convert_to_numpy=True)

                    # Prepare properties
                    properties = {
                        "content": chunk,
                        "metadata": {
                            **doc.metadata,
                            "chunk_index": i,
                            "total_chunks": len(chunks),
                            "original_doc_id": doc.id,
                        },
                        "namespace": request.namespace,
                        "created_at": datetime.utcnow().isoformat(),
                    }

                    # Add to batch
                    batch.add_data_object(
                        properties,
                        "Document",
                        vector=embedding.tolist(),
                    )

                    chunk_count += 1

                ingested_count += 1

        logger.info(f"Ingested {ingested_count} documents ({chunk_count} chunks)")

        return {
            "success": True,
            "documents_ingested": ingested_count,
            "chunks_created": chunk_count,
            "namespace": request.namespace,
        }

    except Exception as e:
        logger.error(f"Error ingesting documents: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/tools/search_similar", response_model=List[SearchResult])
async def search_similar(request: SearchRequest):
    """Semantic search using vector similarity."""
    logger.info(f"Searching for: {request.query[:50]}... (top_k={request.top_k})")

    try:
        # Generate query embedding
        query_vector = embedding_model.encode(request.query, convert_to_numpy=True)

        # Build Weaviate query
        where_filter = {
            "path": ["namespace"],
            "operator": "Equal",
            "valueString": request.namespace,
        }

        # Additional filters
        if request.filters:
            # Combine filters (simplified - should support complex filters)
            pass

        # Execute search
        result = (
            weaviate_client.query
            .get("Document", ["content", "metadata", "namespace"])
            .with_near_vector({"vector": query_vector.tolist()})
            .with_where(where_filter)
            .with_limit(request.top_k)
            .with_additional(["distance", "id"])
            .do()
        )

        # Parse results
        documents = result.get("data", {}).get("Get", {}).get("Document", [])

        search_results = []
        for doc in documents:
            # Convert distance to similarity score (0-1)
            distance = doc["_additional"]["distance"]
            score = 1 - (distance / 2)  # Cosine distance to similarity

            if score >= request.min_score:
                search_results.append(SearchResult(
                    id=doc["_additional"]["id"],
                    content=doc["content"],
                    score=score,
                    metadata=doc.get("metadata", {}),
                ))

        logger.info(f"Found {len(search_results)} results")
        return search_results

    except Exception as e:
        logger.error(f"Error searching: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/tools/hybrid_search", response_model=List[SearchResult])
async def hybrid_search(request: HybridSearchRequest):
    """Hybrid search combining semantic and keyword matching."""
    logger.info(f"Hybrid search (alpha={request.alpha}): {request.query[:50]}...")

    try:
        # Generate query embedding
        query_vector = embedding_model.encode(request.query, convert_to_numpy=True)

        # Build hybrid query
        where_filter = {
            "path": ["namespace"],
            "operator": "Equal",
            "valueString": request.namespace,
        }

        # Execute hybrid search
        result = (
            weaviate_client.query
            .get("Document", ["content", "metadata"])
            .with_hybrid(
                query=request.query,
                alpha=request.alpha,  # Balance between keyword and vector
                vector=query_vector.tolist(),
            )
            .with_where(where_filter)
            .with_limit(request.top_k)
            .with_additional(["score", "id"])
            .do()
        )

        documents = result.get("data", {}).get("Get", {}).get("Document", [])

        search_results = []
        for doc in documents:
            score = doc["_additional"].get("score", 0.0)

            if score >= request.min_score:
                search_results.append(SearchResult(
                    id=doc["_additional"]["id"],
                    content=doc["content"],
                    score=score,
                    metadata=doc.get("metadata", {}),
                ))

        logger.info(f"Found {len(search_results)} hybrid results")
        return search_results

    except Exception as e:
        logger.error(f"Error in hybrid search: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/tools/agentic_retrieval")
async def agentic_retrieval(request: AgenticRetrievalRequest):
    """Multi-step retrieval with iterative reasoning (Agentic RAG)."""
    logger.info(f"Agentic retrieval: {request.query} (max_iterations={request.max_iterations})")

    results = []
    queries = [request.query]  # Start with original query
    seen_doc_ids = set()

    try:
        for iteration in range(request.max_iterations):
            logger.info(f"Iteration {iteration + 1}/{request.max_iterations}")

            # Search with current query
            search_req = SearchRequest(
                query=queries[-1],
                top_k=5,
                namespace=request.namespace,
            )
            iteration_results = await search_similar(search_req)

            # Add new unique results
            for result in iteration_results:
                if result.id not in seen_doc_ids:
                    results.append(result)
                    seen_doc_ids.add(result.id)

            # Generate follow-up query using Claude (simplified - should use actual LLM)
            # In production: analyze results, identify gaps, generate refined query
            if iteration < request.max_iterations - 1:
                # Simplified: add context from results to query
                if results:
                    context_snippet = results[0].content[:100]
                    refined_query = f"{request.query} related to {context_snippet}"
                    queries.append(refined_query)
                else:
                    break  # No results, stop iterating

        return {
            "query": request.query,
            "iterations": len(queries),
            "queries_used": queries,
            "total_results": len(results),
            "results": results[:10],  # Return top 10
        }

    except Exception as e:
        logger.error(f"Error in agentic retrieval: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/tools/retrieve_and_generate")
async def retrieve_and_generate(request: RAGRequest):
    """Retrieve relevant context and generate response (full RAG)."""
    logger.info(f"RAG: {request.query[:50]}...")

    try:
        # Step 1: Retrieve relevant documents
        search_results = await search_similar(SearchRequest(
            query=request.query,
            top_k=request.top_k,
            namespace=request.namespace,
        ))

        if not search_results:
            return {
                "query": request.query,
                "answer": "No relevant information found.",
                "sources": [],
            }

        # Step 2: Combine context
        context = "\n\n".join([
            f"[Source {i+1}]: {result.content}"
            for i, result in enumerate(search_results)
        ])

        # Step 3: Generate response using Claude (via API call to another service)
        # In production, this would call Spec-Synthesizer-MCP or a dedicated LLM service
        # For now, return context

        return {
            "query": request.query,
            "context": context,
            "sources": [
                {
                    "id": result.id,
                    "content": result.content[:200] + "...",
                    "score": result.score,
                    "metadata": result.metadata,
                }
                for result in search_results
            ],
            "answer": "Use this context to generate answer via LLM service",
        }

    except Exception as e:
        logger.error(f"Error in RAG: {e}")
        raise HTTPException(status_code=500, detail=str(e))


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=int(os.getenv("PORT", "3005")))
```

**File:** `services/rag-engine-mcp/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python packages
RUN pip install --no-cache-dir -r requirements.txt

# Download embedding model (cache it in image)
RUN python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('BAAI/bge-large-en-v1.5')"

# Copy application
COPY . .

EXPOSE 3005

CMD ["python", "main.py"]
```

---

### Week 3-4: RAG Integration with Existing Services

#### Task 7.3: Enhance Spec-Synthesizer with RAG

**File:** `services/spec-synthesizer-mcp/rag_client.py`

```python
import httpx
from typing import List, Dict, Any

class RAGClient:
    def __init__(self, rag_url: str):
        self.rag_url = rag_url

    async def search(self, query: str, top_k: int = 5, namespace: str = "default") -> List[Dict[str, Any]]:
        """Search for relevant context."""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.rag_url}/tools/search_similar",
                json={
                    "query": query,
                    "top_k": top_k,
                    "namespace": namespace,
                }
            )
            response.raise_for_status()
            return response.json()

    async def ingest_documents(self, documents: List[Dict[str, Any]], namespace: str = "default"):
        """Ingest documents into RAG."""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.rag_url}/tools/ingest_documents",
                json={
                    "documents": documents,
                    "namespace": namespace,
                }
            )
            response.raise_for_status()
            return response.json()
```

**Update:** `services/spec-synthesizer-mcp/main.py`

```python
# Add at top
from rag_client import RAGClient

# Initialize RAG client
RAG_URL = os.getenv("RAG_URL", "http://rag-engine-mcp:3005")
rag_client = RAGClient(RAG_URL)

# Update generate_user_stories to use RAG
@app.post("/tools/generate_user_stories", response_model=GenerateUserStoriesResponse)
async def generate_user_stories(request: GenerateUserStoriesRequest):
    """Generate user stories with RAG context."""

    logger.info(f"Generating {request.num_stories} user stories")

    # Get relevant context from RAG
    rag_results = []
    if request.project_context:
        try:
            rag_results = await rag_client.search(
                query=f"{request.description} {request.project_context}",
                top_k=3,
                namespace="project_knowledge"
            )
            logger.info(f"Retrieved {len(rag_results)} relevant documents from RAG")
        except Exception as e:
            logger.warning(f"RAG search failed: {e}")

    # Build context from RAG results
    rag_context = ""
    if rag_results:
        rag_context = "\n\nRelevant Context:\n" + "\n\n".join([
            f"- {result['content'][:200]}..."
            for result in rag_results
        ])

    prompt = f"""You are a product manager creating user stories for a software project.

Project Description:
{request.description}

{f"Additional Context:\n{request.project_context}" if request.project_context else ""}

{rag_context}

Generate {request.num_stories} user stories..."""

    # Rest of the function remains the same...
```

---

## Month 8: Research Automation

### Week 1-2: SearXNG Deployment & Research-Harvester-MCP

#### Task 8.1: Deploy SearXNG

**File:** `infrastructure/docker/docker-compose.yml` (add)

```yaml
  # SearXNG - Self-hosted search engine
  searxng:
    image: searxng/searxng:latest
    container_name: ecosystem-searxng
    ports:
      - "8888:8080"
    volumes:
      - ./searxng:/etc/searxng
    environment:
      - SEARXNG_BASE_URL=http://localhost:8888/
    networks:
      - ecosystem-network
```

**File:** `infrastructure/docker/searxng/settings.yml`

```yaml
use_default_settings: true
server:
  secret_key: "your-secret-key-here"  # Generate with: openssl rand -hex 32
  limiter: false
  image_proxy: true

search:
  safe_search: 0
  autocomplete: ""
  default_lang: "en"

engines:
  - name: google
    disabled: false
  - name: duckduckgo
    disabled: false
  - name: github
    disabled: false
  - name: stackoverflow
    disabled: false
  - name: arxiv
    disabled: false
```

#### Task 8.2: Research-Harvester-MCP Implementation

**File:** `services/research-harvester-mcp/requirements.txt`

```
fastapi==0.109.0
uvicorn==0.27.0
pydantic==2.6.0
playwright==1.41.0
beautifulsoup4==4.12.3
aiohttp==3.9.1
celery==5.3.4
redis==5.0.1
newspaper3k==0.2.8
python-dateutil==2.8.2
```

**File:** `services/research-harvester-mcp/main.py`

```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel, Field
from typing import List, Dict, Any, Optional, Literal
import aiohttp
from bs4 import BeautifulSoup
from playwright.async_api import async_playwright
import logging
from datetime import datetime
import hashlib
from urllib.parse import urlparse
import os

logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO").upper())
logger = logging.getLogger(__name__)

app = FastAPI(title="Research Harvester MCP")

# Configuration
SEARXNG_URL = os.getenv("SEARXNG_URL", "http://searxng:8080")
RAG_URL = os.getenv("RAG_URL", "http://rag-engine-mcp:3005")
DATAMAN_URL = os.getenv("DATAMAN_URL", "http://dataman-mcp:3001")

# In-memory storage (should use database in production)
research_tasks: Dict[str, Any] = {}


# Models
class WebSearchRequest(BaseModel):
    query: str
    sources: List[str] = ["google", "duckduckgo", "github", "stackoverflow"]
    max_results: int = Field(10, ge=1, le=100)
    language: str = "en"


class Article(BaseModel):
    url: str
    title: str
    content: str
    author: Optional[str] = None
    published_date: Optional[str] = None
    summary: Optional[str] = None
    metadata: Dict[str, Any] = {}


class ExtractArticleRequest(BaseModel):
    url: str
    use_javascript: bool = False  # Use Playwright for JS-heavy sites


class ResearchTaskRequest(BaseModel):
    topic: str
    sources: List[str] = ["web", "github", "arxiv"]
    depth: Literal["quick", "thorough"] = "quick"
    max_articles: int = 10
    auto_store_rag: bool = True


class ResearchTaskStatus(BaseModel):
    task_id: str
    status: Literal["pending", "running", "completed", "failed"]
    progress: int  # 0-100
    articles_found: int
    articles_processed: int
    started_at: datetime
    completed_at: Optional[datetime] = None


@app.get("/")
async def root():
    return {
        "name": "research-harvester-mcp",
        "version": "0.1.0",
        "protocol": "mcp",
        "tools": [
            {
                "name": "web_search",
                "description": "Search the web using SearXNG meta-search",
            },
            {
                "name": "extract_article",
                "description": "Extract article content from a URL",
            },
            {
                "name": "batch_extract",
                "description": "Extract content from multiple URLs",
            },
            {
                "name": "create_research_task",
                "description": "Create a comprehensive research task",
            },
            {
                "name": "get_task_status",
                "description": "Get the status of a research task",
            },
        ],
    }


@app.get("/health")
async def health():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}


@app.post("/tools/web_search")
async def web_search(request: WebSearchRequest):
    """Search the web using SearXNG."""
    logger.info(f"Searching for: {request.query}")

    try:
        async with aiohttp.ClientSession() as session:
            # Build SearXNG query
            params = {
                "q": request.query,
                "format": "json",
                "categories": ",".join(request.sources),
                "lang": request.language,
            }

            async with session.get(f"{SEARXNG_URL}/search", params=params) as response:
                if response.status != 200:
                    raise HTTPException(
                        status_code=response.status,
                        detail="SearXNG search failed"
                    )

                data = await response.json()
                results = data.get("results", [])[:request.max_results]

                # Parse results
                search_results = []
                for result in results:
                    search_results.append({
                        "url": result.get("url"),
                        "title": result.get("title"),
                        "content": result.get("content", ""),
                        "engine": result.get("engine"),
                        "score": result.get("score", 0),
                    })

                logger.info(f"Found {len(search_results)} results")

                return {
                    "query": request.query,
                    "results": search_results,
                    "total": len(search_results),
                }

    except Exception as e:
        logger.error(f"Error searching: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/tools/extract_article", response_model=Article)
async def extract_article(request: ExtractArticleRequest):
    """Extract article content from a URL."""
    logger.info(f"Extracting article from: {request.url}")

    try:
        if request.use_javascript:
            # Use Playwright for JavaScript-heavy sites
            content = await extract_with_playwright(request.url)
        else:
            # Use BeautifulSoup for static sites
            content = await extract_with_bs4(request.url)

        return content

    except Exception as e:
        logger.error(f"Error extracting article: {e}")
        raise HTTPException(status_code=500, detail=str(e))


async def extract_with_bs4(url: str) -> Article:
    """Extract content using BeautifulSoup."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=30)) as response:
            html = await response.text()

    soup = BeautifulSoup(html, 'html.parser')

    # Remove script and style elements
    for element in soup(["script", "style", "nav", "footer", "header"]):
        element.decompose()

    # Extract title
    title_tag = soup.find('title') or soup.find('h1')
    title = title_tag.get_text(strip=True) if title_tag else urlparse(url).path

    # Extract main content (simplified - should use readability)
    content_tags = soup.find_all(['p', 'article', 'main'])
    content = "\n\n".join([tag.get_text(strip=True) for tag in content_tags])

    # Extract metadata
    author = None
    author_meta = soup.find('meta', attrs={'name': 'author'})
    if author_meta:
        author = author_meta.get('content')

    published_date = None
    date_meta = soup.find('meta', attrs={'property': 'article:published_time'})
    if date_meta:
        published_date = date_meta.get('content')

    return Article(
        url=url,
        title=title,
        content=content[:10000],  # Limit to 10k chars
        author=author,
        published_date=published_date,
        metadata={"extraction_method": "beautifulsoup"},
    )


async def extract_with_playwright(url: str) -> Article:
    """Extract content using Playwright (for JavaScript sites)."""
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()

        try:
            await page.goto(url, wait_until="networkidle", timeout=30000)

            # Extract content
            title = await page.title()
            content = await page.evaluate("""
                () => {
                    // Remove unwanted elements
                    ['script', 'style', 'nav', 'footer', 'header'].forEach(tag => {
                        document.querySelectorAll(tag).forEach(el => el.remove());
                    });

                    // Get main content
                    return document.body.innerText;
                }
            """)

            await browser.close()

            return Article(
                url=url,
                title=title,
                content=content[:10000],
                metadata={"extraction_method": "playwright"},
            )

        except Exception as e:
            await browser.close()
            raise e


@app.post("/tools/batch_extract", response_model=List[Article])
async def batch_extract(urls: List[str]):
    """Extract content from multiple URLs."""
    logger.info(f"Batch extracting {len(urls)} URLs")

    articles = []
    for url in urls:
        try:
            article = await extract_article(ExtractArticleRequest(url=url))
            articles.append(article)
        except Exception as e:
            logger.error(f"Failed to extract {url}: {e}")
            continue

    return articles


@app.post("/tools/create_research_task", response_model=Dict[str, str])
async def create_research_task(request: ResearchTaskRequest, background_tasks: BackgroundTasks):
    """Create a comprehensive research task."""
    task_id = f"research-{len(research_tasks) + 1}"

    task = {
        "id": task_id,
        "topic": request.topic,
        "sources": request.sources,
        "depth": request.depth,
        "max_articles": request.max_articles,
        "status": "pending",
        "progress": 0,
        "articles_found": 0,
        "articles_processed": 0,
        "started_at": datetime.utcnow(),
    }
    research_tasks[task_id] = task

    # Execute in background
    background_tasks.add_task(execute_research_task, task_id, request)

    return {"task_id": task_id, "message": "Research task started"}


async def execute_research_task(task_id: str, request: ResearchTaskRequest):
    """Execute a research task."""
    task = research_tasks[task_id]
    task["status"] = "running"

    try:
        all_articles = []

        # Web search
        if "web" in request.sources:
            search_results = await web_search(WebSearchRequest(
                query=request.topic,
                max_results=request.max_articles,
            ))

            urls = [r["url"] for r in search_results["results"]]
            task["articles_found"] = len(urls)
            task["progress"] = 30

            # Extract articles
            for i, url in enumerate(urls):
                try:
                    article = await extract_article(ExtractArticleRequest(url=url))
                    all_articles.append(article)
                    task["articles_processed"] = i + 1
                    task["progress"] = 30 + int((i + 1) / len(urls) * 50)
                except Exception as e:
                    logger.error(f"Failed to extract {url}: {e}")

        task["progress"] = 80

        # Store in RAG if requested
        if request.auto_store_rag and all_articles:
            try:
                async with aiohttp.ClientSession() as session:
                    documents = [
                        {
                            "content": f"{article.title}\n\n{article.content}",
                            "metadata": {
                                "url": article.url,
                                "title": article.title,
                                "source": "research",
                                "topic": request.topic,
                            },
                        }
                        for article in all_articles
                    ]

                    async with session.post(
                        f"{RAG_URL}/tools/ingest_documents",
                        json={
                            "documents": documents,
                            "namespace": "research",
                        }
                    ) as response:
                        if response.status == 200:
                            logger.info(f"Stored {len(documents)} articles in RAG")

            except Exception as e:
                logger.error(f"Failed to store in RAG: {e}")

        task["status"] = "completed"
        task["progress"] = 100
        task["completed_at"] = datetime.utcnow()
        task["articles"] = [a.dict() for a in all_articles]

        logger.info(f"Research task {task_id} completed: {len(all_articles)} articles")

    except Exception as e:
        logger.error(f"Research task {task_id} failed: {e}")
        task["status"] = "failed"
        task["error"] = str(e)


@app.get("/tools/get_task_status/{task_id}", response_model=ResearchTaskStatus)
async def get_task_status(task_id: str):
    """Get the status of a research task."""
    if task_id not in research_tasks:
        raise HTTPException(status_code=404, detail="Task not found")

    task = research_tasks[task_id]

    return ResearchTaskStatus(
        task_id=task["id"],
        status=task["status"],
        progress=task["progress"],
        articles_found=task["articles_found"],
        articles_processed=task["articles_processed"],
        started_at=task["started_at"],
        completed_at=task.get("completed_at"),
    )


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=int(os.getenv("PORT", "3006")))
```

---

## Month 9: End-to-End Workflows

### Week 1-2: Complete Workflow Templates

#### Task 9.1: Research → Spec → Code → Test → Deploy Workflow

**File:** `workflows/research-to-deploy.yaml`

```yaml
name: Research to Deployment
description: Complete automation from business idea to deployed feature
version: 1.0

parameters:
  - name: business_requirement
    type: string
    description: High-level business requirement or feature request
    required: true
  - name: project_name
    type: string
    description: Name of the project/repository
    required: true
  - name: deploy_environment
    type: string
    enum: [dev, staging, prod]
    default: dev

steps:
  # Step 1: Research
  - name: research
    service: research-harvester-mcp
    tool: create_research_task
    inputs:
      topic: "{{ parameters.business_requirement }}"
      sources: ["web", "github"]
      depth: "thorough"
      max_articles: 10
      auto_store_rag: true
    outputs:
      - task_id

  - name: wait_for_research
    service: research-harvester-mcp
    tool: get_task_status
    inputs:
      task_id: "{{ steps.research.outputs.task_id }}"
    retry:
      max_attempts: 20
      delay_seconds: 30
      until: "status == 'completed'"

  # Step 2: Generate Specification
  - name: generate_user_stories
    service: spec-synthesizer-mcp
    tool: generate_user_stories
    inputs:
      description: "{{ parameters.business_requirement }}"
      project_context: "Research completed, see RAG for details"
      num_stories: 5
    outputs:
      - stories

  - name: generate_technical_design
    service: spec-synthesizer-mcp
    tool: generate_technical_design
    inputs:
      user_stories: "{{ steps.generate_user_stories.outputs.stories }}"
    outputs:
      - design

  - name: generate_api_contract
    service: spec-synthesizer-mcp
    tool: generate_api_contract
    inputs:
      design_doc: "{{ steps.generate_technical_design.outputs.design }}"
      api_style: "rest"
    outputs:
      - contract

  # Step 3: Generate Code
  - name: generate_scaffold
    service: codesmith-mcp
    tool: generate_project_scaffold
    inputs:
      spec:
        projectName: "{{ parameters.project_name }}"
        description: "{{ parameters.business_requirement }}"
        techStack:
          language: "typescript"
          framework: "express"
      gitRepoUrl: "{{ config.git_base_url }}/{{ parameters.project_name }}"
    outputs:
      - project_path
      - branch

  - name: generate_api_code
    service: codesmith-mcp
    tool: generate_api_routes
    inputs:
      api_spec: "{{ steps.generate_api_contract.outputs.contract }}"
      project_path: "{{ steps.generate_scaffold.outputs.project_path }}"
    outputs:
      - files_generated

  # Step 4: Run Tests
  - name: create_sandbox
    service: sandbox-runner-mcp
    tool: create_sandbox
    inputs:
      image: "node:20"
      resources:
        cpu: 2
        memory: "4GB"
      network:
        internet_access: true
    outputs:
      - sandbox_id

  - name: run_tests
    service: sandbox-runner-mcp
    tool: run_tests
    inputs:
      sandbox_id: "{{ steps.create_sandbox.outputs.sandbox_id }}"
      test_framework: "jest"
      test_path: "{{ steps.generate_scaffold.outputs.project_path }}"
    outputs:
      - test_results

  - name: destroy_sandbox
    service: sandbox-runner-mcp
    tool: destroy_sandbox
    inputs:
      sandbox_id: "{{ steps.create_sandbox.outputs.sandbox_id }}"

  # Step 5: Create Pull Request
  - name: create_pr
    service: codesmith-mcp
    tool: create_pull_request
    inputs:
      repo: "{{ parameters.project_name }}"
      branch: "{{ steps.generate_scaffold.outputs.branch }}"
      spec:
        title: "Implement: {{ parameters.business_requirement }}"
        description: |
          ## Automated Implementation

          **Research:** {{ steps.research.outputs.task_id }}
          **User Stories:** {{ len(steps.generate_user_stories.outputs.stories) }} stories
          **Test Results:** {{ steps.run_tests.outputs.test_results.passed }}/{{ steps.run_tests.outputs.test_results.total }} passed

          This PR was automatically generated by the ecosystem automation pipeline.
    outputs:
      - pr_url

  # Step 6: Deploy (if tests passed)
  - name: deploy
    service: devops-conductor-mcp
    tool: deploy
    condition: "{{ steps.run_tests.outputs.test_results.all_passed }}"
    inputs:
      service_name: "{{ parameters.project_name }}"
      version: "{{ steps.generate_scaffold.outputs.branch }}"
      environment: "{{ parameters.deploy_environment }}"
      strategy:
        type: "rolling"
        rollback_on_failure: true
    outputs:
      - deployment_id

on_failure:
  - notify:
      channel: "slack"
      message: "Workflow failed at step {{ failed_step }}: {{ error_message }}"

on_success:
  - notify:
      channel: "slack"
      message: |
        ✅ Successfully deployed {{ parameters.business_requirement }}
        📄 PR: {{ steps.create_pr.outputs.pr_url }}
        🚀 Deployment: {{ steps.deploy.outputs.deployment_id }}
```

---

### Week 3-4: Performance Optimization & Testing

#### Task 9.2: Caching Strategy

**File:** `services/shared/cache_manager.py`

```python
import redis
import json
import hashlib
from typing import Any, Optional
import pickle

class CacheManager:
    def __init__(self, redis_url: str):
        self.redis = redis.from_url(redis_url)

    def _make_key(self, prefix: str, identifier: str) -> str:
        """Create a cache key."""
        hash_id = hashlib.md5(identifier.encode()).hexdigest()
        return f"{prefix}:{hash_id}"

    async def get(self, prefix: str, identifier: str) -> Optional[Any]:
        """Get value from cache."""
        key = self._make_key(prefix, identifier)
        value = self.redis.get(key)
        if value:
            return pickle.loads(value)
        return None

    async def set(self, prefix: str, identifier: str, value: Any, ttl: int = 3600):
        """Set value in cache with TTL."""
        key = self._make_key(prefix, identifier)
        self.redis.setex(key, ttl, pickle.dumps(value))

    async def delete(self, prefix: str, identifier: str):
        """Delete value from cache."""
        key = self._make_key(prefix, identifier)
        self.redis.delete(key)

# Usage example
cache = CacheManager("redis://redis:6379")

# Cache embeddings
embedding = await cache.get("embedding", text)
if embedding is None:
    embedding = model.encode(text)
    await cache.set("embedding", text, embedding, ttl=86400)  # 24 hours
```

---

## RAG Best Practices

### 1. Chunking Strategies

```python
# Semantic chunking (better than fixed-size)
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", ". ", " ", ""],
)

chunks = splitter.split_text(document)
```

### 2. Query Enhancement

```python
# Expand query with synonyms/related terms
async def enhance_query(query: str) -> str:
    # Use LLM to generate query variations
    prompt = f"Generate 3 alternative phrasings of this query: {query}"
    alternatives = await llm.generate(prompt)
    return f"{query} OR {alternatives}"
```

### 3. Re-ranking Results

```python
# Re-rank search results for better relevance
from sentence_transformers import CrossEncoder

reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

def rerank_results(query: str, results: List[SearchResult]) -> List[SearchResult]:
    pairs = [[query, result.content] for result in results]
    scores = reranker.predict(pairs)

    # Sort by re-ranking score
    ranked = sorted(zip(results, scores), key=lambda x: x[1], reverse=True)
    return [r for r, _ in ranked]
```

---

## Performance Optimization

### Metrics to Track

1. **RAG Performance:**
   - Embedding generation time: < 100ms per document
   - Search latency: < 200ms for top-10 results
   - Cache hit rate: > 80%

2. **Research Performance:**
   - Web search: < 2 seconds
   - Article extraction: < 5 seconds per URL
   - Full research task: < 5 minutes for 10 articles

3. **End-to-End Workflow:**
   - Simple feature (CRUD): < 10 minutes
   - Medium feature (API + tests): < 30 minutes
   - Complex feature: < 2 hours

### Optimization Techniques

```python
# 1. Batch operations
async def batch_embed(texts: List[str]) -> List[np.ndarray]:
    return embedding_model.encode(texts, batch_size=32, show_progress_bar=True)

# 2. Parallel execution
import asyncio

results = await asyncio.gather(
    search_web(query),
    search_github(query),
    search_arxiv(query),
)

# 3. Connection pooling
session = aiohttp.ClientSession(
    connector=aiohttp.TCPConnector(limit=100)
)
```

---

## Success Criteria for Phase 3

✅ **RAG Engine:**
- Vector database operational (Weaviate)
- Embedding generation working (< 100ms per doc)
- Semantic search functional (< 200ms)
- Hybrid search implemented
- Agentic retrieval working
- 1000+ documents indexed

✅ **Research Automation:**
- SearXNG deployed and accessible
- Research-Harvester-MCP operational
- Web search working (multiple sources)
- Article extraction functional (static + JS sites)
- Automatic RAG ingestion
- Research tasks complete in < 5 minutes

✅ **End-to-End Workflows:**
- "Research → Deploy" workflow functional
- All steps execute successfully
- Error handling and retries working
- Workflow takes < 30 minutes for simple feature

✅ **Integration:**
- Spec generation uses RAG context
- Code generation uses codebase context
- All services communicate properly
- Performance targets met

---

## Next Steps After Phase 3

1. **Collect usage metrics** - Analyze workflow performance
2. **Refine RAG** - Improve retrieval quality based on usage
3. **Begin Phase 4** - Polish, team features, scaling
4. **Expand workflows** - Add more automation templates
5. **Fine-tune models** - Train local models on your domain

---

**Document Version:** 1.0
**Last Updated:** 2025-11-14
**Next Review:** After completing Month 7
