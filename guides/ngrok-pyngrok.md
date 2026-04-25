# Guia de uso do ngrok com pyngrok em Python

## O que é o ngrok

ngrok é um serviço de **tunneling reverso**: ele cria uma conexão segura entre um servidor rodando localmente (ex.: `localhost:8000`) e um endpoint público na internet (ex.: `https://abc123.ngrok-free.app`), sem necessidade de configurar roteadores, abrir portas ou fazer deploy.

O fluxo é:

```
Internet → ngrok cloud → tunnel criptografado → ngrok agent (local) → localhost:8000
```

O **ngrok agent** é o processo que roda na sua máquina e mantém a conexão com a nuvem do ngrok. `pyngrok` gerencia esse processo automaticamente — você não precisa instalar nem executar o binário `ngrok` manualmente.

---

## Conceitos

### Tunnel

É o canal criado entre o agent local e a nuvem do ngrok. Cada chamada a `ngrok.connect()` abre um tunnel. Um tunnel tem:

- Uma **URL pública** (gerada pelo ngrok) que recebe as requisições
- Uma **porta local** de destino (onde sua aplicação está ouvindo)
- Um **protocolo** (HTTP, HTTPS, TCP ou TLS)

No plano gratuito, a URL pública é gerada aleatoriamente e muda a cada vez que o tunnel é aberto.

### Agent

O processo do ngrok que roda localmente. É ele que mantém a conexão persistente com os servidores do ngrok na nuvem. `pyngrok` baixa e gerencia o binário do agent automaticamente na primeira execução.

### Authtoken

Token que autentica o agent na sua conta ngrok. Sem ele, o ngrok funciona em modo anônimo, com limitações severas (sessão curta, sem HTTPS, conexões simultâneas limitadas). Com o authtoken:

- Sessão sem expiração
- HTTPS automático com certificado válido
- Mais conexões simultâneas
Obtido em: `https://dashboard.ngrok.com/get-started/your-authtoken`

### Endpoint / URL pública

A URL que o ngrok expõe para a internet. Formato no plano free:

```
https://<hash-aleatório>.ngrok-free.app
```

Qualquer requisição HTTP/HTTPS feita a essa URL é encaminhada pelo tunnel para `localhost:<porta>`.

### Domínio fixo (Reserved Domain)

Recurso pago. Permite fixar um subdomínio (ex.: `minha-api.ngrok.app`) que não muda entre restarts. Indispensável quando um serviço externo precisa de uma URL permanente configurada manualmente (ex.: webhook registrado num painel de terceiros).

### Região

Os servidores do ngrok estão distribuídos geograficamente. Por padrão ele escolhe a região mais próxima. Pode ser configurado:

```python
conf.get_default().region = "sa"  # South America
# Outras: "us", "eu", "ap", "au", "jp", "in"
```

### Protocolo HTTP vs TCP

- **HTTP/HTTPS** (padrão): para servidores web. O ngrok entende o protocolo e consegue inspecionar as requisições no dashboard.
- **TCP**: para qualquer outro protocolo (bancos de dados, SSH, sockets). Não há inspeção de tráfego, apenas encaminhamento de bytes.

```python
# HTTP (padrão)
tunnel = ngrok.connect(8000)

# TCP (ex.: expor um banco PostgreSQL local)
tunnel = ngrok.connect(5432, proto="tcp")
```

---

## Instalação

```toml
# pyproject.toml
dependencies = [
    "pyngrok==8.0.0",
]
```

```bash
uv add pyngrok
# ou
pip install pyngrok
```

---

## Configuração mínima

```python
from pyngrok import ngrok, conf

conf.get_default().auth_token = "SEU_NGROK_AUTHTOKEN"
tunnel = ngrok.connect(8000)          # porta local que será exposta
print(tunnel.public_url)              # ex.: https://abc123.ngrok-free.app
```

---

## Padrão recomendado: abrir apenas em `dev`

Guarda o tunnel condicionalmente ao ambiente. Evita abrir tunnel em produção.

```python
import os
from pyngrok import ngrok, conf

authtoken = None
if os.getenv("ENVIRONMENT") == "dev":
    authtoken = os.getenv("NGROK_AUTHTOKEN")
    if authtoken:
        conf.get_default().auth_token = authtoken
        tunnel = ngrok.connect(8000)
        print(f"ngrok tunnel: {tunnel.public_url}")
```

---

## Integração com lifespan do FastAPI

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from pyngrok import ngrok, conf
import os

@asynccontextmanager
async def lifespan(app: FastAPI):
    authtoken = None
    if os.getenv("ENVIRONMENT") == "dev":
        authtoken = os.getenv("NGROK_AUTHTOKEN")
        if authtoken:
            conf.get_default().auth_token = authtoken
            tunnel = ngrok.connect(8000)
            print(f"ngrok tunnel: {tunnel.public_url}")

    yield  # aplicação rodando

    if authtoken:
        ngrok.kill()   # fecha o tunnel ao desligar

app = FastAPI(lifespan=lifespan)
```

---

## Variáveis de ambiente

```bash
# .env.example
NGROK_AUTHTOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ENVIRONMENT=dev   # "dev" abre tunnel, "prod" não abre
```

---

## Principais chamadas da API `pyngrok`

| Chamada | O que faz |
|---|---|
| `conf.get_default().auth_token = token` | Define o authtoken antes de conectar |
| `conf.get_default().region = "sa"` | Define a região dos servidores ngrok |
| `ngrok.connect(port)` | Abre tunnel HTTP para a porta local; retorna objeto `NgrokTunnel` |
| `ngrok.connect(port, proto="tcp")` | Abre tunnel TCP para a porta local |
| `tunnel.public_url` | URL pública gerada (ex.: `https://abc123.ngrok-free.app`) |
| `ngrok.get_tunnels()` | Lista todos os tunnels abertos |
| `ngrok.disconnect(url)` | Fecha um tunnel específico pela URL pública |
| `ngrok.kill()` | Encerra todos os tunnels e o processo ngrok |

---

## Casos de uso típicos

- **Webhooks locais**: expor endpoint local para receber callbacks de serviços externos (Twilio, Stripe, GitHub, etc.) sem deploy
- **Demos rápidas**: compartilhar servidor local temporariamente
- **Testes de integração**: testar integrações que exigem URL pública acessível
- **Expor banco de dados local**: via tunnel TCP, sem abrir portas no roteador

---

## Pontos de atenção

- O authtoken é necessário para tunnels persistentes e HTTPS. Sem ele, o tunnel cai após alguns minutos no plano gratuito.
- `ngrok.kill()` deve ser chamado no shutdown para não deixar processos órfãos.
- O `public_url` pode mudar a cada restart (plano free). Para URL fixa, use domínios reservados no plano pago.
- Em containers Docker, o ngrok deve rodar dentro do container ou em um sidecar separado; `ngrok.connect(8000)` assume que a porta está no mesmo host.
