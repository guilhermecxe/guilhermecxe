# Arquitetura do Projeto

Baseada no padrão **Ports & Adapters (Hexagonal Architecture)**, organizada para separar claramente regras de negócio, integrações externas e contratos de interface.

---

## Estrutura de Pastas

```
project/
├── src/
│   ├── main.py
│   ├── settings.py
│   ├── di.py
│   │
│   ├── routes/
│   │   ├── documents.py
│   │   └── search.py
│   │
│   ├── schemas/
│   │   ├── document.py
│   │   └── search.py
│   │
│   ├── models/
│   │   ├── document.py
│   │   ├── search.py
│   │   └── agent_run.py          ← must-have quando há agentes
│   │
│   ├── services/
│   │   ├── indexing.py
│   │   ├── retrieval.py
│   │   └── agent_run.py          ← must-have quando há agentes
│   │
│   ├── agents/
│   │   ├── research_agent.py
│   │   ├── tools/
│   │   │   ├── retrieval_toolkit.py
│   │   │   └── summarize_toolkit.py
│   │   └── prompts/
│   │       ├── research.py
│   │       └── summarize.py
│   │
│   ├── ports/
│   │   ├── vector_store.py
│   │   ├── embedding_model.py
│   │   ├── document_loader.py
│   │   └── agent_run_store.py    ← must-have quando há agentes
│   │
│   ├── adapters/
│   │   ├── chroma.py
│   │   ├── mongo.py
│   │   ├── openai_embeddings.py
│   │   ├── pdf_loader.py
│   │   └── file_storage.py
│   │
│   └── exceptions.py
│
└── tests/
    ├── services/
    │   ├── test_indexing.py
    │   └── test_retrieval.py
    ├── adapters/
    │   ├── test_chroma.py
    │   └── test_pdf_loader.py
    └── agents/
        └── test_research_agent.py
```

---

## Arquivos Raiz de `src/`

### `main.py`
Ponto de entrada da aplicação. Instancia o framework (FastAPI, Flask etc.), registra as rotas e inicializa o servidor.

**Depende de:** `routes/`, `di.py`, `settings.py`

```python
from fastapi import FastAPI
from src.routes import documents, search
from src.di import container

app = FastAPI()
app.include_router(documents.router)
app.include_router(search.router)
```

---

### `settings.py`
Centraliza toda a configuração da aplicação via variáveis de ambiente. Nenhum outro módulo lê `os.environ` diretamente — todos consomem `settings`.

**Depende de:** nada interno

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    chroma_host: str = "localhost"
    chroma_port: int = 8000
    openai_api_key: str
    vector_store_backend: str = "chroma"  # "chroma" | "mongo"

    class Config:
        env_file = ".env"

settings = Settings()
```

---

### `di.py`
Dependency Injection — o único lugar onde adaptadores são instanciados e conectados aos serviços. Toda a "fiação" da aplicação acontece aqui.

**Depende de:** `settings.py`, `adapters/`, `services/`, `ports/`

```python
from src.settings import settings
from src.adapters.chroma import ChromaVectorStore
from src.adapters.mongo import MongoVectorStore
from src.adapters.openai_embeddings import OpenAIEmbeddingModel
from src.services.indexing import IndexingService
from src.services.retrieval import RetrievalService

def build_vector_store():
    if settings.vector_store_backend == "mongo":
        return MongoVectorStore(uri=settings.mongo_uri)
    return ChromaVectorStore(host=settings.chroma_host, port=settings.chroma_port)

vector_store = build_vector_store()
embedding_model = OpenAIEmbeddingModel(api_key=settings.openai_api_key)

indexing_service = IndexingService(
    vector_store=vector_store,
    embedding_model=embedding_model,
)
retrieval_service = RetrievalService(
    vector_store=vector_store,
    embedding_model=embedding_model,
)
```

---

### `exceptions.py`
Erros de domínio da aplicação. Evita que exceções de bibliotecas externas vazem para as camadas superiores.

**Depende de:** nada interno

```python
class DocumentNotFoundError(Exception):
    pass

class IndexingError(Exception):
    pass

class EmbeddingError(Exception):
    pass
```

---

## `routes/`

Interface HTTP da aplicação. Valida entrada com schemas, chama services, retorna schemas de resposta. Não contém lógica de negócio.

**Depende de:** `schemas/`, `di.py` (via injeção de dependência do framework)

```python
# routes/search.py
from fastapi import APIRouter, Depends
from src.schemas.search import SearchRequest, SearchResponse
from src.di import retrieval_service

router = APIRouter(prefix="/search")

@router.post("/", response_model=SearchResponse)
async def search(request: SearchRequest):
    results = retrieval_service.search(query=request.query, top_k=request.top_k)
    return SearchResponse(results=results)
```

---

## `schemas/`

Contratos de entrada e saída da API. São modelos Pydantic usados exclusivamente na camada de rotas para validação e serialização.

**Depende de:** nada interno  
**Usado por:** `routes/`

```python
# schemas/search.py
from pydantic import BaseModel

class SearchRequest(BaseModel):
    query: str
    top_k: int = 5

class SearchResponse(BaseModel):
    results: list[dict]


# schemas/document.py
from pydantic import BaseModel

class IngestRequest(BaseModel):
    file_path: str
    metadata: dict = {}

class IngestResponse(BaseModel):
    document_id: str
    chunks_indexed: int
```

---

## `models/`

Entidades canônicas que trafegam entre as camadas internas (services, adapters, agents). São a "língua comum" da aplicação — independentes de framework e de biblioteca externa.

**Depende de:** nada interno  
**Usado por:** `services/`, `adapters/`, `agents/`

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
from dataclasses import dataclass

@dataclass
class SearchResult:
    chunk_id: str
    content: str
    score: float
    metadata: dict
```

---

## `ports/`

Interfaces abstratas (Protocols) que definem os contratos que os adapters devem cumprir. Services e agents dependem apenas dessas interfaces, nunca de implementações concretas.

**Depende de:** `models/`  
**Usado por:** `services/`, `agents/`, `di.py`, `adapters/` (como contrato a cumprir)

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
from typing import Protocol

class EmbeddingModel(Protocol):
    def embed(self, text: str) -> list[float]: ...
    def embed_batch(self, texts: list[str]) -> list[list[float]]: ...


# ports/document_loader.py
from typing import Protocol
from src.models.document import Document

class DocumentLoader(Protocol):
    def load(self, path: str) -> Document: ...
```

---

## `adapters/`

Implementações concretas dos ports. Cada arquivo é responsável por uma tecnologia específica. Convertem entre o formato externo da tecnologia e os `models/` internos.

**Depende de:** `ports/` (como contrato), `models/`, `settings.py`  
**Usado por:** `di.py`

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
                results["ids"][0],
                results["documents"][0],
                results["distances"][0],
                results["metadatas"][0],
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
        response = self._client.embeddings.create(input=text, model=self._model)
        return response.data[0].embedding

    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        response = self._client.embeddings.create(input=texts, model=self._model)
        return [item.embedding for item in response.data]


# adapters/pdf_loader.py
import pdfplumber
from src.models.document import Document
import uuid

class PDFLoader:
    def load(self, path: str) -> Document:
        with pdfplumber.open(path) as pdf:
            content = "\n".join(
                page.extract_text() for page in pdf.pages if page.extract_text()
            )
        return Document(id=str(uuid.uuid4()), content=content, metadata={"source": path})
```

---

## `services/`

Regras de negócio da aplicação. Orquestram adapters (via ports) para realizar operações significativas para o sistema. Não sabem qual tecnologia está sendo usada por baixo.

**Depende de:** `ports/`, `models/`, `exceptions.py`  
**Usado por:** `routes/`, `agents/`

```python
# services/indexing.py
from src.ports.vector_store import VectorStore
from src.ports.embedding_model import EmbeddingModel
from src.ports.document_loader import DocumentLoader
from src.models.document import Chunk
from src.exceptions import IndexingError
import uuid

class IndexingService:
    def __init__(
        self,
        vector_store: VectorStore,
        embedding_model: EmbeddingModel,
    ):
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
            Chunk(
                id=str(uuid.uuid4()),
                document_id=document_id,
                content=" ".join(words[i:i + size]),
                metadata={},
            )
            for i in range(0, len(words), size)
        ]


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
```

---

### `services/agent_run.py` ⚠️ must-have quando há agentes

Responsável pelo ciclo de vida de uma execução de agente — início, registro de cada tool call e encerramento (com sucesso ou falha). É o único lugar que persiste o que o agente fez e por quê, viabilizando auditoria, retry e observabilidade.

Sem esse service, execuções de agentes são caixas-pretas: não há como saber o que foi feito, em qual ordem, nem o que falhou.

**Depende de:** `ports/agent_run_store.py`, `models/agent_run.py`  
**Usado por:** agentes (diretamente, não via tool)

```python
# models/agent_run.py
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class AgentStep:
    tool: str
    input: dict
    output: dict
    timestamp: datetime = field(default_factory=datetime.utcnow)

@dataclass
class AgentRun:
    id: str
    agent: str
    input: dict
    status: str                        # "running" | "completed" | "failed"
    steps: list[AgentStep] = field(default_factory=list)
    output: dict | None = None
    error: str | None = None
    started_at: datetime = field(default_factory=datetime.utcnow)
    finished_at: datetime | None = None


# ports/agent_run_store.py
from typing import Protocol
from src.models.agent_run import AgentRun, AgentStep

class AgentRunStore(Protocol):
    def save(self, run: AgentRun) -> None: ...
    def get(self, run_id: str) -> AgentRun: ...
    def append_step(self, run_id: str, step: AgentStep) -> None: ...
    def update_status(self, run_id: str, status: str, **kwargs) -> None: ...


# services/agent_run.py
from src.ports.agent_run_store import AgentRunStore
from src.models.agent_run import AgentRun, AgentStep
from src.exceptions import AgentRunNotFoundError
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
        step = AgentStep(tool=tool, input=input, output=output)
        self._store.append_step(run_id, step)

    def complete(self, run_id: str, output: dict) -> None:
        self._store.update_status(
            run_id, status="completed", output=output, finished_at=datetime.utcnow()
        )

    def fail(self, run_id: str, error: str) -> None:
        self._store.update_status(
            run_id, status="failed", error=error, finished_at=datetime.utcnow()
        )
```

---

## `agents/`

Módulos com certa autonomia — tomam decisões sobre quais tools chamar e em qual ordem, geralmente orientados por um LLM. Cada agente tem seus próprios prompts e ferramentas.

Todo agente deve usar o `AgentRunService` para registrar sua execução. Essa é a separação central: **o agente decide e rastreia, o service executa e persiste.**

**Depende de:** `services/`, `models/`, `agents/tools/`, `agents/prompts/`  
**Usado por:** `routes/` (para endpoints que expõem agentes)

### `agents/tools/`

Tools são organizadas em **toolkits** — classes que funcionam como interface de um service para o agente. A relação é sempre 1-para-1: cada toolkit tem um service equivalente, e cada tool exposta pelo toolkit tem um método ou recurso equivalente nesse service.

```
RetrievalService       ←→    RetrievalToolkit
  .search()            ←→      search_documents (tool)
```

Essa separação existe porque o service serve a toda a aplicação (routes, outros services, workers), enquanto o toolkit adapta a mesma lógica para a interface de tool calling — adicionando docstrings descritivas para o LLM, convertendo tipos de retorno para dict e registrando o passo no `AgentRunService`.

Cada toolkit expõe um método `get_tools()` que retorna a lista de `StructuredTool` prontos para serem passados ao agente.

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
            query: Texto da busca. Deve ser uma pergunta ou frase descritiva
                sobre o conteúdo desejado.
            top_k: Número máximo de resultados a retornar. Padrão: 5.

        Returns:
            Lista de dicionários com os campos ``content`` (texto do chunk)
            e ``score`` (relevância, quanto maior melhor).
        """
        results = self._retrieval.search(query=query, top_k=top_k)
        output = [{"content": r.content, "score": r.score} for r in results]
        if self._run_id:
            self._runs.record_step(
                self._run_id,
                tool="search_documents",
                input={"query": query, "top_k": top_k},
                output=output,
            )
        return output

    def get_tools(self) -> list[StructuredTool]:
        return [
            StructuredTool.from_function(
                func=self._search_documents,
                name="search_documents",
            ),
        ]
```

O agente recebe toolkits, não tools individuais — e injeta o `run_id` antes de iniciar a execução:

```python
# agents/research_agent.py
from src.agents.tools.retrieval_toolkit import RetrievalToolkit
from src.agents.prompts.research import SYSTEM_PROMPT, build_user_prompt
from src.services.agent_run import AgentRunService

class ResearchAgent:
    def __init__(
        self,
        retrieval_toolkit: RetrievalToolkit,
        run_service: AgentRunService,
        llm_client,
    ):
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

No `di.py`, o toolkit recebe o service já instanciado — o service continua sendo a única fonte de lógica, e o toolkit é apenas sua projeção para o mundo do agente:

```python
# di.py (trecho relevante)
from src.agents.tools.retrieval_toolkit import RetrievalToolkit

retrieval_toolkit = RetrievalToolkit(
    retrieval_service=retrieval_service,
    run_service=agent_run_service,
)

research_agent = ResearchAgent(
    retrieval_toolkit=retrieval_toolkit,
    run_service=agent_run_service,
    llm_client=llm_client,
)
```

### `agents/prompts/`
Templates de prompt centralizados por agente. Separa o texto dos prompts da lógica de orquestração.

```python
# agents/prompts/research.py
SYSTEM_PROMPT = """
Você é um assistente de pesquisa. Use as ferramentas disponíveis para
buscar informações relevantes e sintetize uma resposta fundamentada.
Sempre cite as fontes encontradas.
"""

def build_user_prompt(question: str) -> str:
    return f"Responda a seguinte pergunta com base nos documentos disponíveis:\n\n{question}"
```

---

## `tests/`

Espelha a estrutura de `src/`. Com os ports bem definidos, adapters podem ser facilmente mockados nos testes de services e agents.

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

---

## Relações entre Camadas

```
routes/
  └─ valida com ──────────► schemas/
  └─ chama ───────────────► services/   ou   agents/
                                │
                    depende de (via ports/)
                                │
                           adapters/
                                │
                    instanciados em di.py
                    configurados por settings.py

models/  ◄──── usados por todos (schemas convertem para/de models)
ports/   ◄──── contratos que adapters cumprem e services/agents consomem
exceptions/ ◄─ lançadas por adapters, capturadas por services ou routes
```

### Regra de ouro
> Nenhuma camada interna (`services`, `models`, `ports`) importa de `adapters`, `routes` ou `schemas`. O fluxo de dependência aponta sempre para dentro — e `di.py` é o único ponto que conhece tudo.

---

## Adaptações

Variações arquiteturais para suportar execuções assíncronas de agentes. Em ambos os casos, o `AgentRunService` é o mecanismo central — é ele que persiste o estado da execução e torna a resposta assíncrona possível.

O ponto chave é que o consumer **não é um service** — é um entrypoint, análogo ao `main.py`. Ele escuta a fila, delega para o agente, e nenhuma regra de negócio vive nele. Por isso fica em `workers/`, separado de `services/`.

---

### Adaptação: Polling

O cliente enfileira um job, recebe um `run_id` imediatamente e consulta o status periodicamente até a conclusão.

**O que muda na estrutura:**

```
src/
  workers/
    agent_consumer.py     ← entrypoint do processo worker, análogo ao main.py
  services/
    publish.py            ← enfileira o job e inicia o AgentRun
  adapters/
    rabbitmq.py           ← publica e consome mensagens
  ports/
    message_queue.py      ← interface para o broker
  routes/
    jobs.py               ← endpoint GET /jobs/{id}/status
```

**Fluxo:**

```
POST /jobs
  └─ publish_service.enqueue()
      ├─ agent_run_service.start()   → persiste run com status "running", retorna run_id
      └─ rabbitmq.publish(run_id)    → enfileira

GET /jobs/{run_id}/status
  └─ agent_run_service.get(run_id)  → lê status atual do AgentRun
```

```python
# services/publish.py
from src.ports.message_queue import MessageQueue
from src.services.agent_run import AgentRunService
from src.models.agent_run import AgentRun

class PublishService:
    def __init__(self, queue: MessageQueue, run_service: AgentRunService):
        self._queue = queue
        self._runs = run_service

    def enqueue(self, agent: str, input: dict) -> AgentRun:
        run = self._runs.start(agent=agent, input=input)
        self._queue.publish(queue=agent, message={"run_id": run.id, "input": input})
        return run  # run.id é retornado ao cliente para polling


# routes/jobs.py
from fastapi import APIRouter
from src.di import agent_run_service

router = APIRouter(prefix="/jobs")

@router.get("/{run_id}/status")
async def get_status(run_id: str):
    run = agent_run_service.get(run_id)
    return {"status": run.status, "output": run.output, "error": run.error}


# workers/agent_consumer.py
import asyncio
from src.di import rabbitmq, research_agent

async def main():
    async for message in rabbitmq.consume(queue="research"):
        await research_agent.run(
            question=message["input"]["question"],
            run_id=message["run_id"],     # agente usa run_id existente em vez de criar novo
        )

asyncio.run(main())
```

O `AgentRunService` já contém tudo que o endpoint de status precisa — nenhum estado extra é necessário.

---

### Adaptação: Webhook

O cliente enfileira um job informando uma URL de callback. Quando a execução termina, o sistema faz um POST nessa URL com o resultado.

**O que muda na estrutura:**

```
src/
  workers/
    agent_consumer.py     ← mesmo entrypoint, com chamada ao webhook ao final
  services/
    publish.py            ← enfileira o job (igual ao polling)
    webhook.py            ← monta o payload e envia o callback
  adapters/
    rabbitmq.py           ← publica e consome mensagens
    http_client.py        ← executa o POST do callback
  ports/
    message_queue.py      ← interface para o broker
    http_client.py        ← interface para o sender de callbacks
```

**Fluxo:**

```
POST /jobs  (com callback_url no body)
  └─ publish_service.enqueue()
      ├─ agent_run_service.start()
      └─ rabbitmq.publish(run_id + callback_url)

[worker consome]
  └─ research_agent.run()
      └─ ao completar: webhook_service.notify(run, callback_url)
          └─ http_client.post(callback_url, payload)
```

```python
# services/webhook.py
from src.ports.http_client import HttpClient
from src.models.agent_run import AgentRun

class WebhookService:
    def __init__(self, http_client: HttpClient):
        self._client = http_client

    def notify(self, run: AgentRun, callback_url: str) -> None:
        payload = {
            "run_id": run.id,
            "status": run.status,
            "output": run.output,
            "error": run.error,
        }
        self._client.post(url=callback_url, body=payload)


# adapters/http_client.py
import httpx

class HttpxClient:
    def post(self, url: str, body: dict) -> None:
        with httpx.Client() as client:
            client.post(url, json=body, timeout=10)


# ports/http_client.py
from typing import Protocol

class HttpClient(Protocol):
    def post(self, url: str, body: dict) -> None: ...


# workers/agent_consumer.py
import asyncio
from src.di import rabbitmq, research_agent, webhook_service

async def main():
    async for message in rabbitmq.consume(queue="research"):
        await research_agent.run(
            question=message["input"]["question"],
            run_id=message["run_id"],
        )
        run = agent_run_service.get(message["run_id"])
        webhook_service.notify(run=run, callback_url=message["callback_url"])

asyncio.run(main())
```

A diferença em relação ao polling é só o `webhook_service.notify()` ao final do consumer — toda a estrutura de services, adapters e ports permanece a mesma.

---

### Adaptação: Desktop (HTTP → IPC)

Quando o backend serve uma aplicação desktop em vez de clientes HTTP, a camada de entrada muda — mas apenas ela. `routes/` é substituído por `ipc/`, e o broker externo (RabbitMQ) cede lugar a uma fila local. O núcleo da aplicação (`services/`, `agents/`, `ports/`, `adapters/`, `models/`) permanece intacto.

Um exemplo de framework que viabiliza esse padrão com Python no backend é o **Tauri** — ele expõe "comandos" que o frontend (React, Vue etc.) invoca via `invoke()`, e o mecanismo de registro desses comandos é exatamente o que o `main.py` passa a fazer.

**O que sai:**

- `routes/` — não há HTTP, logo não há endpoints REST nem status codes
- RabbitMQ — um broker externo é desnecessário para um processo local; `asyncio.Queue` ou SQLite resolvem
- Variáveis de ambiente no `settings.py` — substituídas por um arquivo de configuração local do usuário

**O que entra:**

```
src/
  ipc/                        ← handlers no lugar de routes/
    documents.py
    search.py
    jobs.py
  adapters/
    local_queue.py            ← fila em memória no lugar do RabbitMQ
```

**O que fica igual:** `services/`, `agents/`, `workers/`, `ports/`, `models/`, `di.py`, `exceptions.py`, `AgentRunService`.

---

**Estrutura de um handler IPC:**

A estrutura interna é quase idêntica a uma rota — recebe payload, valida, chama service, retorna dict. A diferença é que não há HTTP, headers nem status codes. `schemas/` ainda é útil para validar os dados vindos da UI.

```python
# ipc/search.py
from src.di import retrieval_service, publish_service, agent_run_service
from src.schemas.search import SearchRequest

class SearchIPC:
    def handle_search(self, payload: dict) -> dict:
        request = SearchRequest(**payload)
        results = retrieval_service.search(query=request.query, top_k=request.top_k)
        return {"results": [{"content": r.content, "score": r.score} for r in results]}

    def handle_enqueue(self, payload: dict) -> dict:
        run = publish_service.enqueue(agent="research", input=payload)
        return {"run_id": run.id, "status": run.status}

    def handle_job_status(self, payload: dict) -> dict:
        run = agent_run_service.get(payload["run_id"])
        return {"status": run.status, "output": run.output, "error": run.error}
```

**Registro no entrypoint:**

```python
# main.py
from src.ipc.search import SearchIPC
from src.ipc.documents import DocumentsIPC

search_ipc = SearchIPC()
documents_ipc = DocumentsIPC()

app.register_command("search", search_ipc.handle_search)
app.register_command("enqueue_job", search_ipc.handle_enqueue)
app.register_command("job_status", search_ipc.handle_job_status)
app.register_command("ingest_document", documents_ipc.handle_ingest)
```

**Adapter de fila local:**

```python
# adapters/local_queue.py
import asyncio

class LocalQueue:
    def __init__(self):
        self._queues: dict[str, asyncio.Queue] = {}

    def _get(self, queue: str) -> asyncio.Queue:
        if queue not in self._queues:
            self._queues[queue] = asyncio.Queue()
        return self._queues[queue]

    async def publish(self, queue: str, message: dict) -> None:
        await self._get(queue).put(message)

    async def consume(self, queue: str):
        while True:
            yield await self._get(queue).get()
```

O `di.py` passa a instanciar `LocalQueue` no lugar de `RabbitmqAdapter` — nenhuma outra camada percebe a diferença porque ambos cumprem o port `MessageQueue`.

**No frontend (ex: Tauri + TypeScript):**

```typescript
import { invoke } from "@tauri-apps/api"

const results = await invoke("search", { query: "como funciona X?", top_k: 5 })
const job = await invoke("enqueue_job", { question: "resuma o documento" })
const status = await invoke("job_status", { run_id: job.run_id })
```

---

### Adaptação: Pacote Python

Quando a aplicação é distribuída como um pacote Python, não há processo para inicializar nem camada de entrada para expor. O que hoje são entrypoints e configuração de processo viram **API pública do pacote** — o chamador importa, instancia e orquestra.

**O que sai:**

- `main.py` — não há processo para inicializar
- `routes/` e `ipc/` — não há camada de entrada HTTP nem IPC
- `schemas/` — validação de entrada passa a ser responsabilidade do chamador, ou pode ser mantida se o pacote quiser ser defensivo
- `workers/` — quem usa o pacote decide como orquestrar execuções assíncronas
- `di.py` — a montagem das dependências passa para o chamador via `factory.py`

**O que entra:**

```
src/
  __init__.py       ← exporta a API pública do pacote
  factory.py        ← funções de conveniência para instanciar os componentes principais
```

**O que fica igual:** `services/`, `agents/`, `ports/`, `adapters/`, `models/`, `exceptions.py`, `AgentRunService`.

---

**`factory.py`** assume o papel do `di.py`, mas em vez de montar tudo automaticamente na inicialização, oferece funções que o chamador usa para montar o que precisar. Cada função é uma forma conveniente de obter um componente já configurado com os adapters padrão:

```python
# factory.py
from src.services.retrieval import RetrievalService
from src.services.indexing import IndexingService
from src.agents.research_agent import ResearchAgent
from src.adapters.chroma import ChromaVectorStore
from src.adapters.openai_embeddings import OpenAIEmbeddingModel
from src.adapters.pdf_loader import PDFLoader

def create_retrieval_service(
    openai_api_key: str,
    chroma_host: str = "localhost",
    chroma_port: int = 8000,
) -> RetrievalService:
    return RetrievalService(
        vector_store=ChromaVectorStore(host=chroma_host, port=chroma_port),
        embedding_model=OpenAIEmbeddingModel(api_key=openai_api_key),
    )

def create_indexing_service(
    openai_api_key: str,
    chroma_host: str = "localhost",
    chroma_port: int = 8000,
) -> IndexingService:
    return IndexingService(
        vector_store=ChromaVectorStore(host=chroma_host, port=chroma_port),
        embedding_model=OpenAIEmbeddingModel(api_key=openai_api_key),
    )
```

**`__init__.py`** decide o que é público. A regra é exportar: as factories, os services, os models e os ports. Adapters concretos ficam disponíveis mas não em destaque — o chamador os usa só se quiser trazer o próprio:

```python
# __init__.py
from src.factory import create_retrieval_service, create_indexing_service
from src.services.retrieval import RetrievalService
from src.services.indexing import IndexingService
from src.models.document import Document, Chunk
from src.models.search import SearchResult
from src.ports.vector_store import VectorStore
from src.ports.embedding_model import EmbeddingModel
from src.exceptions import IndexingError, EmbeddingError, DocumentNotFoundError
```

**Uso pelo chamador — caminho feliz com factory:**

```python
from meu_pacote import create_retrieval_service

service = create_retrieval_service(openai_api_key="sk-...")
results = service.search(query="como funciona X?")
```

**Uso pelo chamador — trazendo adapter próprio:**

O fato de os ports estarem exportados no `__init__.py` é o que torna o pacote extensível. O chamador pode trazer qualquer implementação que cumpra o contrato:

```python
from meu_pacote import RetrievalService, VectorStore
from minha_infra import PineconeAdapter   # implementa VectorStore

service = RetrievalService(
    vector_store=PineconeAdapter(...),
    embedding_model=...,
)
```

Esse padrão é o mesmo que bibliotecas como LangChain e LlamaIndex adotam com suas abstrações de `VectorStore` e `Embeddings` — a arquitetura de ports já prepara o pacote para isso sem nenhuma mudança adicional.
