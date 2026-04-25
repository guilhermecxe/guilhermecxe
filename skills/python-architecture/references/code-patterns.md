# Padrões de Código por Camada

Implementações modelo. Use como base ao gerar código novo ou revisar existente.

---

## `models/`

```python
# models/document.py
from dataclasses import dataclass

@dataclass
class Document:
    id: str
    content: str
    metadata: dict

@dataclass
class Chunk:
    id: str
    document_id: str
    content: str
    embedding: list[float] | None = None

# models/search.py
@dataclass
class SearchResult:
    chunk_id: str
    content: str
    score: float
    metadata: dict
```

**Regras:** dataclasses puras. Sem imports de libraries externas. Sem métodos de negócio.

---

## `ports/`

```python
# ports/vector_store.py
from typing import Protocol
from src.models.document import Chunk
from src.models.search import SearchResult

class VectorStore(Protocol):
    def upsert(self, chunk: Chunk) -> None: ...
    def query(self, vector: list[float], top_k: int) -> list[SearchResult]: ...
    def delete(self, chunk_id: str) -> None: ...

# ports/embedding_model.py
class EmbeddingModel(Protocol):
    def embed(self, text: str) -> list[float]: ...
    def embed_batch(self, texts: list[str]) -> list[list[float]]: ...

# ports/document_loader.py
from src.models.document import Document

class DocumentLoader(Protocol):
    def load(self, path: str) -> Document: ...

# ports/agent_run_store.py  (must-have com agentes)
from src.models.agent_run import AgentRun, AgentStep

class AgentRunStore(Protocol):
    def save(self, run: AgentRun) -> None: ...
    def get(self, run_id: str) -> AgentRun: ...
    def append_step(self, run_id: str, step: AgentStep) -> None: ...
    def update_status(self, run_id: str, status: str, **kwargs) -> None: ...
```

**Regras:** só `Protocol` com type hints. Sem implementação. Imports apenas de `models/`.

---

## `adapters/`

```python
# adapters/chroma.py
import chromadb
from src.models.document import Chunk
from src.models.search import SearchResult

class ChromaVectorStore:
    def __init__(self, host: str, port: int):
        self._client = chromadb.HttpClient(host=host, port=port)
        self._collection = self._client.get_or_create_collection("documents")

    def upsert(self, chunk: Chunk) -> None:
        self._collection.upsert(
            ids=[chunk.id],
            embeddings=[chunk.embedding],
            documents=[chunk.content],
            metadatas=[chunk.metadata],
        )

    def query(self, vector: list[float], top_k: int) -> list[SearchResult]:
        results = self._collection.query(query_embeddings=[vector], n_results=top_k)
        return [
            SearchResult(chunk_id=id, content=doc, score=dist, metadata=meta)
            for id, doc, dist, meta in zip(
                results["ids"][0], results["documents"][0],
                results["distances"][0], results["metadatas"][0],
            )
        ]

    def delete(self, chunk_id: str) -> None:
        self._collection.delete(ids=[chunk_id])

# adapters/openai_embeddings.py
from openai import OpenAI

class OpenAIEmbeddingModel:
    def __init__(self, api_key: str, model: str = "text-embedding-3-small"):
        self._client = OpenAI(api_key=api_key)
        self._model = model

    def embed(self, text: str) -> list[float]:
        return self._client.embeddings.create(input=text, model=self._model).data[0].embedding

    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        return [i.embedding for i in
                self._client.embeddings.create(input=texts, model=self._model).data]

# adapters/pdf_loader.py
import pdfplumber, uuid
from src.models.document import Document

class PDFLoader:
    def load(self, path: str) -> Document:
        with pdfplumber.open(path) as pdf:
            content = "\n".join(
                p.extract_text() for p in pdf.pages if p.extract_text()
            )
        return Document(id=str(uuid.uuid4()), content=content, metadata={"source": path})
```

**Regras:** implementa o port correspondente. Converte formato externo ↔ `models/`.
Não importa de `services/` nem de outros adapters.

---

## `services/`

```python
# services/retrieval.py
from src.ports.vector_store import VectorStore
from src.ports.embedding_model import EmbeddingModel
from src.models.search import SearchResult

class RetrievalService:
    def __init__(self, vector_store: VectorStore, embedding_model: EmbeddingModel):
        self._store = vector_store
        self._embeddings = embedding_model

    def search(self, query: str, top_k: int = 5) -> list[SearchResult]:
        vector = self._embeddings.embed(query)
        return self._store.query(vector=vector, top_k=top_k)

# services/indexing.py
from src.ports.vector_store import VectorStore
from src.ports.embedding_model import EmbeddingModel
from src.ports.document_loader import DocumentLoader
from src.models.document import Chunk
from src.exceptions import IndexingError
import uuid

class IndexingService:
    def __init__(self, vector_store: VectorStore, embedding_model: EmbeddingModel):
        self._store = vector_store
        self._embeddings = embedding_model

    def index_document(self, document_path: str, loader: DocumentLoader) -> int:
        try:
            document = loader.load(document_path)
            chunks = self._chunk(document.content, document_id=document.id)
            for chunk in chunks:
                chunk.embedding = self._embeddings.embed(chunk.content)
                self._store.upsert(chunk)
            return len(chunks)
        except Exception as e:
            raise IndexingError(f"Failed to index {document_path}") from e

    def _chunk(self, content: str, document_id: str, size: int = 500) -> list[Chunk]:
        words = content.split()
        return [
            Chunk(id=str(uuid.uuid4()), document_id=document_id,
                  content=" ".join(words[i:i + size]), metadata={})
            for i in range(0, len(words), size)
        ]

# services/agent_run.py  (must-have com agentes)
from src.ports.agent_run_store import AgentRunStore
from src.models.agent_run import AgentRun, AgentStep
import uuid
from datetime import datetime

class AgentRunService:
    def __init__(self, store: AgentRunStore):
        self._store = store

    def start(self, agent: str, input: dict) -> AgentRun:
        run = AgentRun(id=str(uuid.uuid4()), agent=agent, input=input, status="running")
        self._store.save(run)
        return run

    def record_step(self, run_id: str, tool: str, input: dict, output: dict) -> None:
        self._store.append_step(run_id, AgentStep(tool=tool, input=input, output=output))

    def complete(self, run_id: str, output: dict) -> None:
        self._store.update_status(run_id, status="completed",
                                  output=output, finished_at=datetime.utcnow())

    def fail(self, run_id: str, error: str) -> None:
        self._store.update_status(run_id, status="failed",
                                  error=error, finished_at=datetime.utcnow())
```

**Regras:** recebe ports via `__init__`. Lança exceções de domínio (`exceptions.py`).
Nunca importa de `adapters/` ou `routes/`.

---

## `routes/`

```python
# routes/search.py
from fastapi import APIRouter
from src.schemas.search import SearchRequest, SearchResponse
from src.di import retrieval_service

router = APIRouter(prefix="/search")

@router.post("/", response_model=SearchResponse)
async def search(request: SearchRequest):
    results = retrieval_service.search(query=request.query, top_k=request.top_k)
    return SearchResponse(results=results)
```

**Regras:** valida com schemas, chama service, retorna schema. Sem lógica de negócio.
Não importa de `ports/` ou `adapters/` diretamente.

---

## `agents/tools/` — Toolkit Pattern

Relação 1-para-1 com o service equivalente. O toolkit adapta o service para o contexto
de tool calling: adiciona docstrings descritivas para o LLM, converte retornos para `dict`,
registra cada passo no `AgentRunService`.

```python
# agents/tools/retrieval_toolkit.py
from langchain_core.tools import StructuredTool
from src.services.retrieval import RetrievalService
from src.services.agent_run import AgentRunService

class RetrievalToolkit:
    def __init__(self, retrieval_service: RetrievalService, run_service: AgentRunService):
        self._retrieval = retrieval_service
        self._runs = run_service
        self._run_id: str | None = None

    def set_run_id(self, run_id: str) -> None:
        self._run_id = run_id

    def _search_documents(self, query: str, top_k: int = 5) -> list[dict]:
        """Busca chunks relevantes na base de conhecimento dado uma query.

        Args:
            query: Texto da busca — pergunta ou frase descritiva.
            top_k: Número máximo de resultados. Padrão: 5.

        Returns:
            Lista de dicts com ``content`` (texto) e ``score`` (relevância).
        """
        results = self._retrieval.search(query=query, top_k=top_k)
        output = [{"content": r.content, "score": r.score} for r in results]
        if self._run_id:
            self._runs.record_step(self._run_id, tool="search_documents",
                                   input={"query": query, "top_k": top_k}, output=output)
        return output

    def get_tools(self) -> list[StructuredTool]:
        return [StructuredTool.from_function(func=self._search_documents,
                                             name="search_documents")]
```

---

## Agente

```python
# agents/research_agent.py
from src.agents.tools.retrieval_toolkit import RetrievalToolkit
from src.agents.prompts.research import SYSTEM_PROMPT, build_user_prompt
from src.services.agent_run import AgentRunService

class ResearchAgent:
    def __init__(self, retrieval_toolkit: RetrievalToolkit,
                 run_service: AgentRunService, llm_client):
        self._retrieval_toolkit = retrieval_toolkit
        self._runs = run_service
        self._llm = llm_client

    def run(self, question: str) -> str:
        run = self._runs.start(agent="research", input={"question": question})
        self._retrieval_toolkit.set_run_id(run.id)
        tools = self._retrieval_toolkit.get_tools()
        try:
            answer = self._llm.chat_with_tools(
                system=SYSTEM_PROMPT,
                user=build_user_prompt(question),
                tools=tools,
            )
            self._runs.complete(run.id, output={"answer": answer})
            return answer
        except Exception as e:
            self._runs.fail(run.id, error=str(e))
            raise
```

---

## `di.py`

```python
from src.settings import settings
from src.adapters.chroma import ChromaVectorStore
from src.adapters.mongo import MongoVectorStore
from src.adapters.openai_embeddings import OpenAIEmbeddingModel
from src.services.indexing import IndexingService
from src.services.retrieval import RetrievalService
from src.services.agent_run import AgentRunService
from src.agents.tools.retrieval_toolkit import RetrievalToolkit
from src.agents.research_agent import ResearchAgent

def build_vector_store():
    if settings.vector_store_backend == "mongo":
        return MongoVectorStore(uri=settings.mongo_uri)
    return ChromaVectorStore(host=settings.chroma_host, port=settings.chroma_port)

vector_store = build_vector_store()
embedding_model = OpenAIEmbeddingModel(api_key=settings.openai_api_key)
indexing_service = IndexingService(vector_store=vector_store, embedding_model=embedding_model)
retrieval_service = RetrievalService(vector_store=vector_store, embedding_model=embedding_model)
# agent_run_service = AgentRunService(store=<adapter>)
# retrieval_toolkit = RetrievalToolkit(retrieval_service=retrieval_service, run_service=agent_run_service)
# research_agent = ResearchAgent(retrieval_toolkit=retrieval_toolkit, run_service=agent_run_service, llm_client=llm)
```

**Regras:** único arquivo que instancia adapters e conecta tudo. Todo mundo importa daqui,
não de `adapters/` diretamente.

---

## `tests/`

```python
# tests/services/test_retrieval.py
from unittest.mock import MagicMock
from src.services.retrieval import RetrievalService
from src.models.search import SearchResult

def test_search_returns_results():
    mock_store = MagicMock()
    mock_embeddings = MagicMock()
    mock_embeddings.embed.return_value = [0.1, 0.2, 0.3]
    mock_store.query.return_value = [
        SearchResult(chunk_id="1", content="resultado", score=0.95, metadata={})
    ]

    service = RetrievalService(vector_store=mock_store, embedding_model=mock_embeddings)
    results = service.search(query="como funciona X?", top_k=3)

    assert len(results) == 1
    assert results[0].content == "resultado"
    mock_embeddings.embed.assert_called_once_with("como funciona X?")
    mock_store.query.assert_called_once_with(vector=[0.1, 0.2, 0.3], top_k=3)
```

**Regras:** espelha `src/`. Mocka ports — nunca adapters concretos — para testar services.
