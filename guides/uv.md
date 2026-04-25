# Guia de uso do UV

UV é um gerenciador de pacotes e projetos Python ultrarrápido, escrito em Rust. Substitui `pip`, `pip-tools`, `virtualenv`, `pyenv` e `poetry` em um único binário.

---

## Instalação

```shell
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# via pip (caso já tenha Python)
pip install uv
```

---

## Inicializar um projeto

```shell
# Cria pyproject.toml, .python-version e estrutura básica
uv init meu-projeto

# Ou inicializa no diretório atual
uv init
```

---

## Gerenciar a versão do Python

UV pode instalar e gerenciar versões do Python sem ferramentas externas.

```shell
# Instalar uma versão específica
uv python install 3.12

# Definir a versão usada no projeto (gera .python-version)
uv python pin 3.12

# Listar versões disponíveis para instalação
uv python list
```

A restrição de versão também pode ser declarada no `pyproject.toml`:

```toml
[project]
requires-python = ">=3.12"
```

---

## Gerenciar dependências

```shell
# Adicionar um pacote (atualiza pyproject.toml e uv.lock)
uv add fastapi

# Adicionar com versão fixa
uv add "fastapi==0.135.3"

# Adicionar dependência de desenvolvimento (grupo dev)
uv add --dev pytest

# Remover um pacote
uv remove requests

# Sincronizar o ambiente com o lock file (instala tudo)
uv sync

# Sincronizar sem dependências de desenvolvimento
uv sync --no-dev

# Atualizar todos os pacotes para as versões mais recentes permitidas
uv lock --upgrade

# Atualizar apenas um pacote específico
uv lock --upgrade-package fastapi
```

---

## Executar comandos no ambiente do projeto

UV detecta e usa automaticamente o virtualenv do projeto.

```shell
# Rodar um script
uv run python script.py

# Rodar qualquer binário instalado no projeto
uv run uvicorn src.main:app --host 0.0.0.0 --port 8000

# Abrir o shell com o ambiente ativado
uv shell
```

> Você raramente precisa ativar o virtualenv manualmente — `uv run` cuida disso.

---

## Índices customizados de pacotes

Útil quando um pacote está em um índice alternativo ao PyPI (ex.: versão CPU do PyTorch).

```toml
# pyproject.toml

[[tool.uv.index]]
name = "pytorch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true  # só usa esse índice quando explicitamente referenciado

[tool.uv.sources]
torch = { index = "pytorch-cpu" }
```

Com `explicit = true`, os demais pacotes continuam sendo resolvidos pelo PyPI normalmente.

---

## Workspaces (monorepo)

UV suporta workspaces para projetos com múltiplos subpacotes Python no mesmo repositório.

```toml
# pyproject.toml na raiz do repositório
[tool.uv.workspace]
members = ["api", "workers"]
```

Cada membro tem seu próprio `pyproject.toml`. O `uv.lock` fica na raiz e resolve tudo junto.

Para adicionar um pacote a um membro específico:

```shell
uv add --project api fastapi
```

---

## Lock file (`uv.lock`)

O `uv.lock` garante builds reproduzíveis — registra versões exatas e hashes de todos os pacotes.

- **Sempre versione o `uv.lock`** junto com o código.
- Gerado/atualizado automaticamente por `uv add`, `uv remove` e `uv lock`.
- Para regenerar do zero: `uv lock`.

---

## Uso em Dockerfile

UV publica imagens base oficiais com Python e UV já instalados.

```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim

WORKDIR /app

# Copia apenas os arquivos de dependência para aproveitar o cache de camadas
COPY pyproject.toml uv.lock ./

# Instala dependências (sem o grupo dev) usando cache do BuildKit
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --no-dev

# Copia o restante do código
COPY . .

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Boas práticas no Docker:**

- Copie `pyproject.toml` e `uv.lock` **antes** do código-fonte para maximizar o cache de camadas. Se o código mudar mas as dependências não, a camada de `uv sync` é reutilizada.
- Use `--mount=type=cache,target=/root/.cache/uv` para evitar baixar pacotes novamente a cada build.
- Use `--no-dev` para não instalar dependências de desenvolvimento na imagem de produção.
- Use `uv run` no `CMD` em vez de ativar o virtualenv manualmente.

---

## Referência rápida de comandos

| Comando | O que faz |
|---|---|
| `uv init` | Inicializa um novo projeto |
| `uv python install 3.12` | Instala uma versão do Python |
| `uv python pin 3.12` | Fixa a versão do Python no projeto |
| `uv add <pacote>` | Adiciona dependência |
| `uv add --dev <pacote>` | Adiciona dependência de desenvolvimento |
| `uv remove <pacote>` | Remove dependência |
| `uv sync` | Sincroniza o ambiente com o lock file |
| `uv sync --no-dev` | Sincroniza sem dependências dev |
| `uv lock` | Regenera o lock file |
| `uv lock --upgrade` | Atualiza todas as dependências |
| `uv run <comando>` | Executa um comando no ambiente do projeto |
| `uv shell` | Abre shell com ambiente ativado |

---

## Recursos oficiais

- Documentação: https://docs.astral.sh/uv
- Repositório: https://github.com/astral-sh/uv
