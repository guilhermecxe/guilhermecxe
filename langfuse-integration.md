# Integrando Langfuse em Projetos Python

Guia prático para adicionar observabilidade com Langfuse em aplicações Python.

---

## Sumário

1. [Conceito](#conceito)
2. [Cenário A — Langfuse em servidor externo](#cenário-a--langfuse-em-servidor-externo)
3. [Cenário B — Langfuse como dependência do projeto (self-hosted)](#cenário-b--langfuse-como-dependência-do-projeto-self-hosted)
4. [Integração com LangChain / LangGraph](#integração-com-langchain--langgraph)

---

## Conceito

O Langfuse funciona como um servidor de observabilidade (traces, spans, métricas de LLM). Sua aplicação se comunica com ele via SDK, enviando eventos assíncrona e automaticamente. Para isso, três variáveis de ambiente são suficientes:

```env
LANGFUSE_PUBLIC_KEY=pk-...
LANGFUSE_SECRET_KEY=sk-...
LANGFUSE_BASE_URL=http://<host>:3000   # padrão: https://cloud.langfuse.com
```

O SDK lê essas variáveis automaticamente — não é preciso passá-las explicitamente ao instanciar clientes.

---

## Cenário A — Langfuse em servidor externo

Use quando já existe um servidor Langfuse rodando (cloud ou servidor interno da equipe).

### 1. Instalar o SDK

```bash
# uv
uv add langfuse

# pip
pip install langfuse
```

### 2. Configurar as variáveis de ambiente

No `.env` (nunca suba para o repositório):

```env
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_BASE_URL=https://langfuse.sua-empresa.com   # ou https://cloud.langfuse.com
```

### 3. Inicializar o cliente na aplicação

Em aplicações FastAPI, o padrão recomendado é inicializar no `lifespan` e fazer `flush` no encerramento:

```python
# main.py
from contextlib import asynccontextmanager
from langfuse import get_client
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.langfuse_client = get_client()
    yield
    app.state.langfuse_client.flush()   # garante envio de eventos pendentes

app = FastAPI(lifespan=lifespan)
```

> `get_client()` retorna um singleton thread-safe. Chamadas subsequentes retornam o mesmo objeto.

### 4. Injetar o cliente via DI (opcional)

```python
# di.py
from langfuse import Langfuse
from fastapi import Request

def get_langfuse_client(request: Request) -> Langfuse:
    return request.app.state.langfuse_client
```

---

## Cenário B — Langfuse como dependência do projeto (self-hosted)

Use quando o servidor Langfuse precisa rodar junto com a aplicação, sem depender de infraestrutura externa.

### Estrutura de arquivos

```
projeto/
├── langfuse/               # subdiretório com o docker-compose oficial do Langfuse
│   └── docker-compose.yml
├── docker-compose.yml      # sua aplicação
├── Makefile
└── api/
    └── .env
```

### 1. Obter o docker-compose do Langfuse

Clone o repositório oficial ou copie apenas o `docker-compose.yml`:

```bash
git clone https://github.com/langfuse/langfuse.git langfuse
```

O `langfuse/docker-compose.yml` sobe os seguintes serviços:

| Serviço | Função |
|---|---|
| `langfuse-web` | Interface web (porta 3000) |
| `langfuse-worker` | Processamento de eventos (porta 3030) |
| `postgres` | Metadados e configurações |
| `clickhouse` | Armazenamento de traces |
| `redis` | Filas internas |
| `minio` | Armazenamento de objetos (eventos/mídia) |

### 2. Usar uma rede Docker compartilhada

A aplicação e o Langfuse precisam se comunicar via rede. O padrão recomendado é uma rede externa nomeada:

```yaml
# docker-compose.yml (aplicação)
networks:
  default:
    name: meu-projeto-network
    external: true
```

```yaml
# langfuse/docker-compose.yml (ao final do arquivo)
networks:
  default:
    name: meu-projeto-network
    external: true
```

### 3. Makefile para orquestrar os dois compose

```makefile
NETWORK = meu-projeto-network
LANGFUSE = meu-projeto-langfuse
API = meu-projeto-api

network:
    docker network inspect $(NETWORK) >/dev/null 2>&1 || \
    docker network create $(NETWORK)

build: network
    docker-compose -p $(LANGFUSE) -f langfuse/docker-compose.yml up -d --build
    docker-compose -p $(API) -f docker-compose.yml up -d --build

up: network
    docker-compose -p $(LANGFUSE) -f langfuse/docker-compose.yml up -d
    docker-compose -p $(API) -f docker-compose.yml up -d

down:
    docker-compose -p $(LANGFUSE) -f langfuse/docker-compose.yml down
    docker-compose -p $(API) -f docker-compose.yml down
```

### 4. Configurar as variáveis de ambiente

Com o Langfuse rodando localmente:

```env
LANGFUSE_PUBLIC_KEY=pk-lf-...     # gerado na UI do Langfuse (localhost:3000)
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_BASE_URL=http://langfuse-web:3000   # nome do serviço Docker
```

> Acesse `http://localhost:3000` para criar um projeto e gerar as chaves.

### 5. Senhas padrão do docker-compose

O `docker-compose.yml` oficial usa valores padrão marcados com `# CHANGEME`. Para produção, substitua antes de subir:

- `SALT`, `ENCRYPTION_KEY`, `NEXTAUTH_SECRET`
- `POSTGRES_PASSWORD`, `CLICKHOUSE_PASSWORD`
- `REDIS_AUTH`, `MINIO_ROOT_PASSWORD`

---

## Integração com LangChain / LangGraph

O Langfuse fornece um `CallbackHandler` nativo para LangChain/LangGraph que captura automaticamente todos os passos de execução (prompts, respostas, tool calls, latência, tokens).

### Dependência adicional

Não há pacote extra — o `langfuse` já inclui a integração:

```python
from langfuse.langchain import CallbackHandler
```

### Uso básico

O cliente Langfuse deve ser inicializado antes (ex.: no `lifespan` do FastAPI, conforme passo 3 do Cenário A). O `CallbackHandler` pode ser instanciado sem argumentos pois lê as variáveis de ambiente automaticamente:

```python
from langfuse.langchain import CallbackHandler

async def ainvoke(self, message: str, ...) -> str:
    input = {"messages": [{"role": "user", "content": message}]}

    callbacks = [CallbackHandler()]   # sem args — usa as env vars

    response = await self._agent.ainvoke(
        input=input,
        config={"callbacks": callbacks}
    )
    return response["messages"][-1].content
```

> O `CallbackHandler` deve ser instanciado a cada chamada (não reutilize entre requests) para que cada invocação gere um trace independente no Langfuse.

### O que é capturado automaticamente

Ao passar o `CallbackHandler` via `config["callbacks"]`, o Langfuse registra:

- Prompt enviado ao LLM e resposta recebida
- Chamadas a tools (nome, input, output)
- Latência de cada passo e do fluxo completo
- Modelo utilizado e contagem de tokens
- Erros e exceções

### Adicionando metadados ao trace (opcional)

Para enriquecer o trace com dados da requisição (ex.: `user_id`, `session_id`):

```python
from langfuse.langchain import CallbackHandler

handler = CallbackHandler(
    user_id="user-123",
    session_id=thread_id,
    metadata={"environment": "production"},
)
```

### Exemplo com LangGraph `create_react_agent`

```python
from langchain.chat_models import init_chat_model
from langchain.agents import create_agent
from langfuse.langchain import CallbackHandler

llm = init_chat_model("openai:gpt-4o-mini")
agent = create_agent(model=llm, tools=tools, system_prompt="...")

response = await agent.ainvoke(
    input={"messages": [{"role": "user", "content": "..."}]},
    config={"callbacks": [CallbackHandler()]}
)
```

### Flush na finalização

O `flush()` no shutdown do FastAPI garante que nenhum evento fique na fila ao encerrar o processo:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.langfuse_client = get_client()
    yield
    app.state.langfuse_client.flush()
```
