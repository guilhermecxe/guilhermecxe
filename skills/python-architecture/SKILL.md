---
name: python-architecture
description: >
  Consultoria de arquitetura para projetos Python.
  Use esta skill sempre que o usuário pedir para criar arquivos, módulos, services, adapters,
  agents, ports, rotas, workers ou qualquer estrutura de código dentro de um projeto
  que segue essa arquitetura. Também use para revisar ou dar manutenção em código existente
  — mover lógica para a camada certa, corrigir imports proibidos, refatorar para respeitar
  os contratos entre camadas. Ative sempre que o usuário mencionar: "onde fica isso?",
  "como adiciono X à arquitetura?", "cria o service/adapter/port/agent para...",
  "refatora isso para a arquitetura", ou qualquer tarefa de scaffolding ou manutenção
  em projetos com pastas src/services, src/ports, src/adapters, src/agents ou src/di.py.
---

# Python Architecture — Skill de Consultoria

Você é um consultor especialista nesta arquitetura. Seu papel é dois:
1. **Criar** estrutura de pastas, arquivos e código seguindo os contratos entre camadas.
2. **Manter** código existente sem desorganizar — mover lógica para o lugar certo, corrigir
   dependências proibidas, sugerir onde cada coisa deve viver.

---

## Mapa das Camadas

```
api/
├── src/
│   ├── main.py        ← entrypoint, monta o framework, registra routes/ipc
│   ├── settings.py    ← config de menor sigilo do que os.environ
│   ├── di.py          ← único lugar que instancia adapters e conecta tudo
|   ├── exceptions.py  ← erros de domínio (sem deps externas)
│   │
│   ├── routes/        ← handlers HTTP; chamam services, retornam schemas
│   ├── schemas/       ← Pydantic para validar entrada/saída da API
│   ├── models/        ← dataclasses puras; a "língua comum" da app
│   ├── services/      ← regras de negócio; orquestram adapters via ports
│   ├── agents/        ← orquestração LLM; tools/ e prompts/ como sub-pastas
│   │   ├── tools/     ← toolkits: projeção de um service para tool calling
│   │   └── prompts/   ← templates de prompt por agente como variáveis constantes
│   ├── ports/         ← Protocols (interfaces); o que adapters devem cumprir
│   └── adapters/      ← implementações concretas de ports (tecnologia)
│
└── tests/
    ├── services/
    ├── adapters/
    └── agents/
```

---

## Regra de Ouro

> **Camadas internas (`models`, `ports`, `services`) nunca importam de camadas externas
> (`adapters`, `routes`, `schemas`, `agents`).  
> O fluxo de dependência aponta sempre para dentro.  
> `di.py` é o único ponto que conhece e instancia tudo.**

### Matriz de dependências permitidas

| Camada       | Pode importar de                          | Nunca importa de              |
|--------------|-------------------------------------------|-------------------------------|
| `models/`    | nada interno                              | tudo                          |
| `exceptions` | nada interno                              | tudo                          |
| `ports/`     | `models/`                                 | `adapters/`, `services/`      |
| `services/`  | `ports/`, `models/`, `exceptions/`        | `adapters/`, `routes/`        |
| `adapters/`  | `ports/`, `models/`, `settings.py`        | `services/`, `routes/`        |
| `schemas/`   | nada interno                              | `models/`, `services/`        |
| `routes/`    | `schemas/`, `di.py`                       | `adapters/`, `ports/` direto  |
| `agents/`    | `services/`, `models/`, `agents/tools/`   | `adapters/`, `routes/`        |
| `workers/`   | `di.py`                                   | lógica de negócio             |
| `di.py`      | tudo                                      | —                             |

---

## Onde Cada Coisa Vive — Árvore de Decisão

**Pergunta 1: É um tipo de dado que circula entre camadas?**
→ `models/` — dataclass pura, sem métodos de negócio.

**Pergunta 2: É uma interface que define o que uma tecnologia deve fazer?**
→ `ports/` — Protocol com type hints. Sem implementação.

**Pergunta 3: É a implementação de uma tecnologia específica (banco, API externa, fila)?**
→ `adapters/` — implementa um port. Converte formato externo ↔ models internos.

**Pergunta 4: É lógica de negócio que orquestra ports para algo significativo?**
→ `services/` — recebe ports via `__init__`. Não sabe qual adapter está por baixo.

**Pergunta 5: É a entrada HTTP que valida, chama service e retorna?**
→ `routes/` para HTTP, `ipc/` para desktop. Não contém lógica de negócio.

**Pergunta 6: É validação de entrada/saída da API?**
→ `schemas/` — Pydantic. Exclusivo para routes/ipc.

**Pergunta 7: É uma ferramenta que o agente pode invocar?**
→ `agents/tools/` como **toolkit** — classe que adapta um service para tool calling.
  - Relação 1-para-1: cada toolkit tem um service equivalente.
  - Expõe `get_tools() → list[StructuredTool]`.

**Pergunta 8: É um consumer de fila/worker assíncrono?**
→ `workers/` — entrypoint puro. Escuta fila, delega para agent. Sem lógica de negócio.

**Pergunta 9: É config da aplicação?**
→ `settings.py` — BaseSettings.
→ `.env`.

---

## Regras de Manutenção

Ao modificar código existente, verifique:

1. **Import proibido?** Se um `service` importa de `adapters/`, mova a dependência para `di.py`
   via injeção no `__init__`.

2. **Lógica de negócio em route/ipc?** Extraia para um service. Route só valida, chama, retorna.

3. **Adapter importando outro adapter?** Errado. Adapters são independentes. Se precisam
   coordenar, crie um service que os orquestre.

4. **Novo port necessário?** Crie o Protocol em `ports/`, implemente o adapter, registre em `di.py`.
   Services/agents passam a receber a interface, não a implementação.

5. **Novo adapter para port existente?** Só adicione o arquivo em `adapters/` e atualize `di.py`
   (geralmente um `if settings.backend == "novo"`). Nenhuma outra camada muda.

---

## Referências Detalhadas

Para código completo e padrões canônicos, leia os arquivos em `references/`:

- **`references/code-patterns.md`** — implementações modelo por camada (models, ports,
  adapters, services, routes, agents/toolkits, di.py, workers, tests)
- **`references/adaptations.md`** — variações arquiteturais: Polling, Webhook, Desktop (IPC),
  Pacote Python — o que entra, sai e fica igual em cada uma

Leia o arquivo relevante antes de gerar código para uma camada ou adaptação específica.
