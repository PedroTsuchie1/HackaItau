# TECH_STACK.md — Itaú-Native Agent Squads (MVP Crédito Agro)

> Stack escolhida para maximizar **confiabilidade da demo** e **velocidade de 3 devs em paralelo**. Cada escolha tem uma alternativa descartada e o motivo. Detalhes de como as peças se encaixam estão em `ARCHITECTURE.md`.

---

## 1. Resumo

| Camada | Escolha | Versão alvo |
|---|---|---|
| Linguagem backend | Python | 3.12 |
| API | FastAPI + Uvicorn | FastAPI ≥ 0.115, Uvicorn ≥ 0.30 |
| Schemas / validação | Pydantic v2 + pydantic-settings | ≥ 2.8 |
| Orquestração | código próprio (`asyncio`) | stdlib |
| LLM | SDK `openai` (endpoint OpenAI-compatível) + `MockProvider` próprio | `openai` ≥ 1.40 |
| RAG | `rank-bm25` sobre chunks Markdown (sem vector DB) | ≥ 0.2.2 |
| Persistência | dict em memória + snapshot JSON em disco | stdlib |
| Frontend | React + TypeScript + Vite | React 18, Vite 5, TS 5 |
| Estilo | Tailwind CSS + `lucide-react` (ícones) | Tailwind 3 |
| Estado no frontend | `useReducer` + hook de polling próprio | — |
| Comunicação FE↔BE | REST JSON + polling de eventos (1 s) | — |
| Testes | `pytest` + `pytest-asyncio` + `httpx` (backend); `vitest` mínimo (frontend) | — |
| Lint/format | `ruff` (backend); `eslint` + `prettier` (frontend) | — |
| Empacotamento | Dockerfile multi-stage (frontend build → backend serve estático) | — |
| Deploy | Render (web service via Dockerfile) — fallback Railway/Fly.io; plano B: Vercel (FE) + Render (BE) | — |
| Gerência de deps | `uv` (ou `pip` + `requirements.txt`) e `npm` | — |

---

## 2. Backend

### 2.1 Python 3.12 + FastAPI + Pydantic v2

- **Por quê**: FastAPI dá OpenAPI grátis (vira `docs/api.md`), tipagem forte via Pydantic, `async` nativo para rodar agentes em paralelo com `asyncio.gather`, e a equipe já domina.
- **Pydantic v2** é o coração dos contratos: `AgentTask`, `AgentOutput`, `ReviewOutput`, `FinalResult`, `Event`, `AuditEvent`, `AgentCard`. O mesmo schema valida output de LLM (`response_format` JSON), a API, e é espelhado em `frontend/src/lib/types.ts`.
- **pydantic-settings** para `Settings` lido de `.env`.

Descartado: Django (pesado), Flask (sem async/tipagem nativa), Node no backend (perderíamos Pydantic como fonte única de contratos).

### 2.2 Orquestração em código próprio

- `orchestrator.py` / `planner.py` / `executor.py` com `asyncio` puro. ~300–400 linhas no total.
- **Descartado**: LangGraph, CrewAI, AutoGen, Semantic Kernel. Motivos: (a) o README exige governança fora do LLM, contexto mínimo controlado e tool gateway próprio — frameworks abstraem justamente isso; (b) curva de aprendizado e bugs de versão no meio de um hackathon; (c) a demo precisa mostrar **nossa** lógica de formação/governança de squads, não a de uma lib.

### 2.3 LLM: SDK `openai` + provider abstrato

```python
class LLMProvider(Protocol):
    async def complete_json(self, *, system, user, schema, model_tier) -> tuple[BaseModel, Usage]: ...
```

- `OpenAICompatProvider`: usa `openai.AsyncOpenAI(base_url=LLM_BASE_URL, api_key=LLM_API_KEY)` com `response_format={"type": "json_schema", ...}` quando suportado, senão `json_object` + validação Pydantic + 1 retry. Funciona com OpenAI, Azure OpenAI, OpenRouter, Groq, Together, Ollama — troca por env.
- `MockProvider`: respostas determinísticas indexadas por `(agent_id, version)`. É o provider padrão em testes e o fallback em `DEMO_MODE`.
- Roteamento de modelos por tier (README §22): `LLM_MODEL_SMALL` (intent/extração) e `LLM_MODEL_STRONG` (análise/review). Podem ser o mesmo modelo se o tempo apertar.
- Opcional: `anthropic` como segundo provider se alguém já tiver chave — mesma interface, arquivo separado, não bloqueia ninguém.

Descartado: LiteLLM (mais uma dependência para o que 60 linhas resolvem); chamadas HTTP cruas (perde retry/tipagem do SDK).

### 2.4 RAG simples: `rank-bm25`

- Docs mock em `backend/app/knowledge/docs/*.md`, chunk por seção (`##`). `retriever.search(query, k=3)` retorna `[{source_id: "POL-CRED-002#1", text, score}]`.
- Índice construído em memória no startup (< 50 ms para ~20 chunks).
- Descartado: Chroma/FAISS/pgvector + embeddings — custo de setup, chamadas extras de embedding, nenhum ganho perceptível com 5 documentos curtos. Se sobrar tempo (P2), trocar a função `score()` por embeddings sem mudar a interface.

### 2.5 Cálculos determinísticos

`tools/calculations.py` em Python puro: `calculate_credit_metrics(financials, request)`, `run_stress_scenarios(base, productivity_baseline, shocks)`. Sem pandas/numpy (não precisamos, e reduz imagem Docker).

### 2.6 Persistência

- `CaseStore`: `dict[str, Case]` + `json.dump` para `backend/runtime/cases/{case_id}.json` em cada transição de estado.
- Descartado: SQLite/Postgres — um caso por demo, sem consultas relacionais; JSON é legível no vídeo e no debug. Se P2 exigir multi-case listável, `sqlite3` da stdlib entra em 30 min.

### 2.7 Telemetria

`services/telemetry.py`: por task de agente registra `latency_ms`, `tokens_in/out` (de `Usage`), `tool_calls`, `model`, `retries`, `fallback_used`; agrega em `case.metrics`. Logging estruturado com `logging` stdlib em JSON (`python-json-logger` opcional). Sem OpenTelemetry/Langfuse no MVP.

---

## 3. Frontend

### 3.1 React 18 + TypeScript + Vite

- **Por quê Vite (SPA) e não Next.js**: não há SSR, SEO, nem rotas server-side; o build estático é servido pelo próprio FastAPI, gerando **um só deploy e um só link**. Vite inicia em < 1 s e tem proxy `/api` nativo para dev.
- TypeScript com `types.ts` espelhando os schemas Pydantic (gerado à mão no dia 1 ou via `openapi-typescript`).

Descartado: Next.js (mais um servidor/deploy para gerenciar, sem benefício aqui), Streamlit (não transmite "banco/seriedade" e limita a UI de cards/timeline).

### 3.2 Tailwind CSS + lucide-react

- Tailwind para velocidade e consistência visual (paleta sóbria: navy/laranja Itaú-like, cinzas). Sem biblioteca de componentes pesada; se quiser acelerar, `shadcn/ui` (copia componentes para o repo, sem runtime extra) é aceitável.
- `lucide-react` para ícones de status (check, alert, shield, lock). Sem avatares de agentes (README §54).

### 3.3 Estado e dados

- `useCaseEvents(caseId)`: hook que faz `GET /api/cases/{id}/events?after=seq` a cada 1 s, aplica no `reducer.ts` e devolve o estado derivado (cards, timeline, banners). Ao receber `HUMAN_REVIEW_REQUIRED` ou `CASE_COMPLETED`, busca `GET /api/cases/{id}` para o resultado consolidado.
- Sem Redux/Zustand/React Query — um recurso, um hook.
- `mocks/events.demo.json`: sequência de eventos gravada que permite desenvolver toda a UI sem backend (`VITE_USE_MOCKS=true`).

Descartado: SSE/WebSocket — polling de 1 s é indistinguível visualmente, funciona atrás de qualquer proxy grátis e não precisa de reconexão. SSE fica como P2 (`GET /api/cases/{id}/stream`).

---

## 4. Testes e qualidade

| Onde | Ferramenta | Mínimo obrigatório (README §52) |
|---|---|---|
| Backend | `pytest`, `pytest-asyncio`, `httpx.AsyncClient` | policy engine (usuário/agente/purpose/no union), outputs seguem schema, evidence_ids presentes, nenhum output aprova crédito, review detecta inconsistência planejada, orquestrador reabre a tarefa certa, caso não termina antes do human gate, `test_demo_case.py` roda o fluxo completo com `MockProvider` |
| Frontend | `vitest` + `@testing-library/react` | reducer aplica sequência `events.demo.json` e termina em `human_review_required` |
| Lint | `ruff check` + `ruff format`; `eslint` + `prettier` | roda no `pre-commit` e no CI |
| CI | GitHub Actions (1 job: lint + pytest + `npm run build`) | verde antes de qualquer deploy |

---

## 5. Empacotamento e deploy

### 5.1 Dockerfile multi-stage

```dockerfile
FROM node:20-alpine AS fe
WORKDIR /fe
COPY frontend/package*.json ./ && RUN npm ci
COPY frontend/ . && RUN npm run build          # → /fe/dist

FROM python:3.12-slim
WORKDIR /app
COPY backend/requirements.txt . && RUN pip install --no-cache-dir -r requirements.txt
COPY backend/ .
COPY --from=fe /fe/dist ./static
ENV DEMO_MODE=true PORT=8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

FastAPI monta `/api/*` e serve `static/` com fallback para `index.html` (SPA).

### 5.2 Hospedagem

- **Primária: Render** (Web Service, Docker, free/starter). Deploy automático do branch `main`. URL pública sem login.
  - Atenção: free tier "dorme" após inatividade → abrir a URL 2 min antes da apresentação, ou usar starter pago no dia.
- **Fallback**: Railway ou Fly.io com o mesmo Dockerfile.
- **Plano B (se o container der problema)**: Vercel para o `frontend/` (com `VITE_API_URL`) + Render para `backend/` (habilitar CORS). Mesmo código, só muda env.
- `docker-compose.yml` para rodar tudo localmente com um comando.

---

## 6. Dev tooling

| Item | Escolha |
|---|---|
| Python deps | `uv` (`uv sync`) — fallback `pip install -r requirements.txt` |
| Node deps | `npm` (lockfile commitado) |
| Pre-commit | `ruff`, `prettier`, `eslint` |
| Editor | qualquer; `.editorconfig` no repo |
| Env | `.env.example` commitado; `.env` gitignored; secrets nunca no código |
| Scripts | `make dev` (backend + frontend com hot reload), `make test`, `make demo` (roda `test_demo_case` e imprime a timeline) |

---

## 7. Variáveis de ambiente (`.env.example`)

```bash
# Modo
DEMO_MODE=true                 # dados fictícios, case pré-carregado, fallback determinístico
DEMO_STEP_DELAY_MS=600         # ritmo visual da timeline quando o MockProvider responde instantâneo
DEMO_USER_ID=USER-DEMO-001

# LLM (opcional; sem chave → MockProvider)
LLM_PROVIDER=openai_compat     # openai_compat | mock
LLM_BASE_URL=https://api.openai.com/v1
LLM_API_KEY=
LLM_MODEL_SMALL=gpt-4o-mini
LLM_MODEL_STRONG=gpt-4o
LLM_TIMEOUT_S=20
LLM_MAX_RETRIES=1

# App
PORT=8000
CORS_ORIGINS=http://localhost:5173
LOG_LEVEL=INFO

# Frontend (build-time)
VITE_API_URL=                  # vazio = mesma origem (deploy único)
VITE_USE_MOCKS=false
```

---

## 8. Como rodar localmente

```bash
# backend
cd backend && uv sync && uv run uvicorn app.main:app --reload --port 8000
# frontend
cd frontend && npm i && npm run dev          # http://localhost:5173 (proxy /api → 8000)
# tudo junto
docker compose up --build                     # http://localhost:8000
# testes
cd backend && uv run pytest -q
```

---

## 9. O que NÃO usamos e por quê (resumo)

| Não usamos | Motivo |
|---|---|
| Microserviços / filas (Redis, RabbitMQ, Celery) | 1 caso, 4 agentes, < 60 s; `asyncio` basta |
| LangGraph / CrewAI / AutoGen | governança e contexto mínimo precisam ser nossos; menos abstração = menos surpresa |
| Vector DB + embeddings | 5 docs curtos; BM25 resolve e é determinístico |
| Postgres / SQLite | JSON em disco é suficiente e legível |
| Next.js | sem SSR; SPA servida pelo backend dá um único deploy |
| WebSocket / SSE | polling de 1 s é suficiente e mais robusto em hosts grátis |
| Auth real (OAuth/JWT) | permissões são mock por `user_id`; link público sem login é requisito |
| OpenTelemetry / Langfuse | telemetria própria em `case.metrics` cobre o que a UI mostra |
| Fine-tuning | fora de escopo (README §23.4) |
