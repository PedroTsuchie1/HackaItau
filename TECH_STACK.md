# TECH_STACK.md — Itaú-Native Agent Squads (MVP)

> Stack escolhida para **confiabilidade da demo e velocidade de 3 devs em paralelo**, não para produção. Cada escolha lista a alternativa rejeitada e o motivo. Referência de arquitetura: `ARCHITECTURE.md`.

---

## 1. Resumo

| Camada | Escolha | Por quê | Rejeitado |
|---|---|---|---|
| Linguagem backend | **Python 3.12** | Ecossistema LLM, Pydantic, asyncio nativo para paralelismo de agentes. | Node (menos maduro para schemas + LLM). |
| Framework API | **FastAPI 0.115+** + **Uvicorn** | Async, OpenAPI automático (gera tipos TS), `BackgroundTasks`/`asyncio.create_task`. | Flask (sem async nativo), Django (peso). |
| Schemas | **Pydantic v2** | Outputs estruturados dos agentes, validação de `evidence_ids`, JSON Schema para `complete_json`. | dataclasses (sem validação). |
| Orquestração | **Código próprio** (`orchestration/`) ~300 linhas | O README exige mostrar a lógica de orquestração/governança; frameworks escondem isso e adicionam risco de versão. | LangGraph, CrewAI, AutoGen (abstração excessiva, difícil impor `ToolGateway`). |
| LLM | **Abstração própria `LLMProvider`** com `openai` SDK (OpenAI-compatible) + `anthropic` SDK + `MockLLMProvider` | Trocar provider por env var; SDK OpenAI cobre OpenAI, Azure OpenAI, Groq, OpenRouter, Ollama. | LiteLLM (dependência extra; a abstração necessária tem 40 linhas). |
| Persistência | **In-memory + snapshot JSON** (`runtime/cases/*.json`) | Demo single-instance; zero setup; reload por arquivo. Interface `CaseStore` permite trocar por SQLite. | SQLite (ok como plano B), Postgres/Redis (infra desnecessária). |
| RAG | **`rank-bm25`** sobre chunks Markdown | Sem embeddings, sem vector DB, determinístico, offline. | Chroma/FAISS + embeddings (custo, latência, mais um provider). |
| Frontend | **React 18 + TypeScript + Vite 5** | SPA estática servida pelo FastAPI → 1 deploy. Build em segundos. | Next.js (SSR desnecessário; 2 deploys ou config extra para servir estático). |
| Estilo | **Tailwind CSS 3** + componentes próprios simples | Visual "banco/sério" rápido; sem design system pesado. | MUI/Ant (peso, aparência genérica). shadcn/ui opcional se sobrar tempo. |
| Data fetching | **TanStack Query v5** (`refetchInterval: 1000`) | Polling incremental de eventos com 5 linhas; cache por `case_id`. | SSE/WebSocket (mais falhas em proxies free-tier). |
| Roteamento | **React Router v6** | `/` e `/cases/:id`. | — |
| Ícones | **lucide-react** | Sem avatares caricatos; ícones neutros. | — |
| Tipos compartilhados | **openapi-typescript** gerando `frontend/src/lib/types.ts` a partir de `/openapi.json` | Contrato único; quebra em build se backend mudar. | Duplicar tipos à mão. |
| Testes backend | **pytest + pytest-asyncio + httpx** (`TestClient`) | Testes do README §52. | — |
| Testes frontend | **Nenhum automatizado** (smoke manual com fixtures) | Tempo de hackathon; contrato é garantido pelos tipos gerados. | Vitest/Playwright (P2). |
| Lint/format | **ruff** (backend), **eslint + prettier** (frontend) | Config padrão, sem discussão. | — |
| Gerência de deps | **uv** (backend), **pnpm** (frontend) | Rápidos e determinísticos (`uv.lock`, `pnpm-lock.yaml`). Fallback: `pip` + `npm`. | poetry (lento). |
| Containers | **Docker** multi-stage + **docker-compose** para dev | Um `Dockerfile` = um artefato de deploy. | — |
| Deploy | **Render (Web Service, Docker)** — 1 serviço público | Link único sem login; Docker nativo; free tier suficiente. Alternativas equivalentes: Fly.io, Railway. | Vercel + Render (2 URLs, CORS, 2 pontos de falha). |
| Observabilidade | **Telemetry própria** em `CaseState.metrics` + `logging` estruturado (JSON) | Tokens/latência por agente aparecem na UI; sem SaaS. | Langfuse/LangSmith (P2). |

---

## 2. Versões fixadas

### Backend (`backend/pyproject.toml`)

```toml
[project]
name = "agent-squads-backend"
requires-python = ">=3.12"
dependencies = [
  "fastapi>=0.115,<0.116",
  "uvicorn[standard]>=0.30,<0.31",
  "pydantic>=2.8,<3",
  "pydantic-settings>=2.4,<3",
  "openai>=1.40,<2",
  "anthropic>=0.34,<1",
  "rank-bm25>=0.2.2,<0.3",
  "python-dotenv>=1.0,<2",
]

[project.optional-dependencies]
dev = ["pytest>=8", "pytest-asyncio>=0.23", "httpx>=0.27", "ruff>=0.5"]
```

### Frontend (`frontend/package.json`, principais)

```json
{
  "dependencies": {
    "react": "^18.3", "react-dom": "^18.3", "react-router-dom": "^6.26",
    "@tanstack/react-query": "^5.51", "lucide-react": "^0.4xx", "clsx": "^2.1"
  },
  "devDependencies": {
    "vite": "^5.4", "typescript": "^5.5", "@vitejs/plugin-react": "^4.3",
    "tailwindcss": "^3.4", "postcss": "^8.4", "autoprefixer": "^10.4",
    "openapi-typescript": "^7.3", "eslint": "^9", "prettier": "^3"
  }
}
```

Node 20 LTS, pnpm 9. Python 3.12, uv 0.4+.

---

## 3. Configuração (`.env.example`)

```bash
# --- Modo ---
DEMO_MODE=true                 # true: case pré-carregado, tools determinísticas, MockLLMProvider por padrão
LLM_FALLBACK_TO_MOCK=true      # se o provider real falhar/timeout, usa mock e segue a demo
MAX_REWORK_LOOPS=2

# --- LLM (ignorado se DEMO_MODE=true e LLM_PROVIDER=mock) ---
LLM_PROVIDER=mock              # mock | openai | anthropic
LLM_BASE_URL=                  # opcional: endpoint OpenAI-compatible (Azure, Groq, OpenRouter, Ollama)
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
LLM_MODEL_FAST=gpt-4o-mini     # roteamento/interpretação/classificação
LLM_MODEL_STRONG=gpt-4o        # Risk, Structuring, Review
LLM_TIMEOUT_SECONDS=20

# --- App ---
APP_ENV=dev                    # dev | prod
CORS_ORIGINS=http://localhost:5173
RUNTIME_DIR=./runtime          # snapshots JSON dos cases
PORT=8000

# --- Frontend (Vite, prefixo obrigatório) ---
VITE_API_BASE_URL=http://localhost:8000
```

Regras: `.env` no `.gitignore`; segredos só em variáveis de ambiente do Render; `DEMO_MODE=true` **é o padrão em produção** (a demo não pode depender do LLM).

---

## 4. Estratégia de modelos (README §22)

| Tarefa | Tier | Justificativa |
|---|---|---|
| Interpretar demanda / extrair `Intent` | `LLM_MODEL_FAST` (ou regex em DEMO_MODE) | Classificação simples, JSON pequeno. |
| Eligibility (checklist) | `FAST` | Regras + lista de pendências; pouco raciocínio. |
| Credit Risk (narrativa sobre métricas calculadas em código) | `STRONG` | Racional de risco e mitigantes. Números vêm das tools. |
| Structuring (alternativas A/B) | `STRONG` | Trade-offs e racional. |
| Review (issues qualitativas além das regras em código) | `STRONG` | Red team. |
| Consolidação do `FinalResult` | **Template/código** | Sem LLM: junta outputs já estruturados. |

Interface:

```python
class LLMProvider(Protocol):
    async def complete_json(self, *, system: str, user: str, schema: type[BaseModel],
                            tier: Literal["fast", "strong"], agent_id: str) -> tuple[BaseModel, Usage]: ...
```

`Usage{input_tokens, output_tokens, latency_ms, model}` alimenta `Telemetry`. Implementações usam *structured outputs* (OpenAI `response_format=json_schema`; Anthropic via tool-use forçado); fallback: parse + retry.

---

## 5. Execução local

```bash
# 1. Backend
cd backend && uv sync --extra dev && cp ../.env.example ../.env
uv run uvicorn app.main:app --reload --port 8000      # http://localhost:8000/docs

# 2. Frontend
cd frontend && pnpm install
pnpm gen:types            # openapi-typescript http://localhost:8000/openapi.json -o src/lib/types.ts
pnpm dev                  # http://localhost:5173 (proxy /api → 8000 via vite.config.ts)

# 3. Tudo junto
docker compose up --build  # backend:8000 + frontend:5173

# 4. Testes / lint
cd backend && uv run pytest -q && uv run ruff check .
cd frontend && pnpm lint && pnpm build
```

---

## 6. Build e deploy

`Dockerfile` (multi-stage):

```dockerfile
FROM node:20-alpine AS web
WORKDIR /web
COPY frontend/ .
RUN corepack enable && pnpm install --frozen-lockfile && pnpm build   # → /web/dist

FROM python:3.12-slim
WORKDIR /app
COPY backend/ .
COPY --from=web /web/dist ./static
RUN pip install uv && uv sync --no-dev --frozen
ENV DEMO_MODE=true PORT=8000
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`main.py` monta `StaticFiles(directory="static", html=True)` em `/` **depois** das rotas `/api` (SPA fallback para `index.html`). Em produção `VITE_API_BASE_URL` fica vazio (mesma origem) → sem CORS.

Render: *New Web Service → Docker → repo → env vars do `.env.example` → Health check `/health`*. Testar o link em aba anônima (checklist do README §3). Antes da apresentação: `curl https://<app>.onrender.com/health` para acordar a instância.

---

## 7. Convenções de código

- **Backend:** `ruff` (line-length 100), type hints obrigatórios, sem `Any` em schemas, módulos em `agents/` não importam `tools/` (teste garante), toda função de tool é pura e síncrona sobre JSON mock (o gateway a envolve em async).
- **Frontend:** componentes funcionais, `types.ts` gerado (não editar), estado de servidor só via TanStack Query, sem lógica de negócio (a UI só renderiza `events`, `CaseState`, `FinalResult`).
- **Dados mock:** todo arquivo em `backend/data/` começa com `"_meta": {"mock": true, "source": "Hackathon demo", "confidential": false}`; nomes de empresas fictícios; valores em `mock_index` quando forem preço.
- **Textos da UI:** os três avisos obrigatórios do README §55 são constantes em `frontend/src/lib/copy.ts`.
- **Commits:** `feat(backend): ...`, `feat(frontend): ...`, `feat(gov): ...`, `docs: ...`, `data: ...`. Branch por dev; PR curto para `main`; `main` deve sempre rodar em `DEMO_MODE`.
