# Adaptações Arquiteturais

Variações do núcleo base para cenários específicos. Em todas as adaptações,
`services/`, `agents/`, `ports/`, `adapters/`, `models/` e `exceptions.py` permanecem intactos.

---

## Adaptação: Polling (Agentes Assíncronos)

O cliente enfileira um job, recebe `run_id` imediatamente e consulta status periodicamente.

**O que entra:**
```
src/
  workers/
    agent_consumer.py     ← entrypoint do processo worker; análogo ao main.py
  services/
    publish.py            ← enfileira job e inicia AgentRun
  adapters/
    rabbitmq.py           ← publica e consome mensagens
  ports/
    message_queue.py      ← interface para o broker
  routes/
    jobs.py               ← GET /jobs/{id}/status
```

**Fluxo:**
```
POST /jobs
  └─ publish_service.enqueue()
      ├─ agent_run_service.start()   → run_id retornado ao cliente
      └─ rabbitmq.publish(run_id)

GET /jobs/{run_id}/status
  └─ agent_run_service.get(run_id)  → lê status atual
```

**Código chave:**
```python
# services/publish.py
class PublishService:
    def __init__(self, queue: MessageQueue, run_service: AgentRunService):
        self._queue = queue
        self._runs = run_service

    def enqueue(self, agent: str, input: dict) -> AgentRun:
        run = self._runs.start(agent=agent, input=input)
        self._queue.publish(queue=agent, message={"run_id": run.id, "input": input})
        return run

# routes/jobs.py
@router.get("/{run_id}/status")
async def get_status(run_id: str):
    run = agent_run_service.get(run_id)
    return {"status": run.status, "output": run.output, "error": run.error}

# workers/agent_consumer.py  ← entrypoint puro, sem lógica de negócio
import asyncio
from src.di import rabbitmq, research_agent

async def main():
    async for message in rabbitmq.consume(queue="research"):
        await research_agent.run(
            question=message["input"]["question"],
            run_id=message["run_id"],
        )

asyncio.run(main())
```

**O que fica igual:** `services/`, `agents/`, `ports/`, `adapters/`, `models/`, `AgentRunService`.

---

## Adaptação: Webhook

Igual ao Polling, com uma chamada extra no worker ao final da execução.

**O que entra (além do Polling):**
```
src/
  services/
    webhook.py            ← monta payload e envia callback
  adapters/
    http_client.py        ← executa o POST
  ports/
    http_client.py        ← interface para o sender de callbacks
```

**Código chave:**
```python
# services/webhook.py
class WebhookService:
    def __init__(self, http_client: HttpClient):
        self._client = http_client

    def notify(self, run: AgentRun, callback_url: str) -> None:
        self._client.post(url=callback_url, body={
            "run_id": run.id, "status": run.status,
            "output": run.output, "error": run.error,
        })

# adapters/http_client.py
import httpx

class HttpxClient:
    def post(self, url: str, body: dict) -> None:
        with httpx.Client() as client:
            client.post(url, json=body, timeout=10)

# ports/http_client.py
class HttpClient(Protocol):
    def post(self, url: str, body: dict) -> None: ...

# workers/agent_consumer.py
async def main():
    async for message in rabbitmq.consume(queue="research"):
        await research_agent.run(question=message["input"]["question"],
                                 run_id=message["run_id"])
        run = agent_run_service.get(message["run_id"])
        webhook_service.notify(run=run, callback_url=message["callback_url"])
```

A única diferença do Polling é o `webhook_service.notify()` ao final do consumer.

---

## Adaptação: Desktop (HTTP → IPC)

Backend serve app desktop (ex: Tauri). `routes/` vira `ipc/`. Broker externo vira fila local.

**O que sai:** `routes/`, RabbitMQ, variáveis de ambiente no `settings.py`.

**O que entra:**
```
src/
  ipc/                        ← handlers no lugar de routes/
    documents.py
    search.py
    jobs.py
  adapters/
    local_queue.py            ← asyncio.Queue no lugar do RabbitMQ
```

**Código chave:**
```python
# ipc/search.py  ← estrutura quase idêntica a uma route; sem HTTP/status codes
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

# main.py
app.register_command("search", search_ipc.handle_search)
app.register_command("enqueue_job", search_ipc.handle_enqueue)

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

`di.py` instancia `LocalQueue` no lugar de `RabbitmqAdapter` — nenhuma outra camada percebe
porque ambos cumprem o port `MessageQueue`.

**Frontend (Tauri + TypeScript):**
```typescript
import { invoke } from "@tauri-apps/api"
const results = await invoke("search", { query: "como funciona X?", top_k: 5 })
const job = await invoke("enqueue_job", { question: "resuma o documento" })
```

---

## Adaptação: Pacote Python

Distribuído como biblioteca. Não há processo para inicializar nem camada de entrada.

**O que sai:** `main.py`, `routes/`, `ipc/`, `schemas/` (opcional), `workers/`, `di.py`.

**O que entra:**
```
src/
  __init__.py       ← exporta a API pública do pacote
  factory.py        ← funções de conveniência para instanciar componentes
```

**`factory.py`** assume o papel do `di.py`, mas expõe funções em vez de montar tudo
na inicialização:

```python
# factory.py
def create_retrieval_service(
    openai_api_key: str,
    chroma_host: str = "localhost",
    chroma_port: int = 8000,
) -> RetrievalService:
    return RetrievalService(
        vector_store=ChromaVectorStore(host=chroma_host, port=chroma_port),
        embedding_model=OpenAIEmbeddingModel(api_key=openai_api_key),
    )
```

**`__init__.py`** exporta factories, services, models e ports. Adapters ficam disponíveis
mas não em destaque — o chamador os usa só se quiser trazer o próprio:

```python
# __init__.py
from src.factory import create_retrieval_service, create_indexing_service
from src.services.retrieval import RetrievalService
from src.models.document import Document, Chunk
from src.models.search import SearchResult
from src.ports.vector_store import VectorStore
from src.ports.embedding_model import EmbeddingModel
from src.exceptions import IndexingError, EmbeddingError
```

**Uso pelo chamador — caminho feliz:**
```python
from meu_pacote import create_retrieval_service
service = create_retrieval_service(openai_api_key="sk-...")
results = service.search(query="como funciona X?")
```

**Uso pelo chamador — adapter próprio:**
```python
from meu_pacote import RetrievalService, VectorStore
from minha_infra import PineconeAdapter  # implementa VectorStore

service = RetrievalService(vector_store=PineconeAdapter(...), embedding_model=...)
```

O fato de os ports estarem exportados é o que torna o pacote extensível — mesmo padrão
de LangChain e LlamaIndex.
