# ARCHITECTURE.md — Itaú-Native Agent Squads (MVP Crédito Agro)

> Documento de arquitetura do MVP de hackathon. Complementa o `README.md` (visão de produto) e é a referência para `TECH_STACK.md` e `TASKS.md`.
> Princípio-guia: **um monólito modular, determinístico o suficiente para uma demo de 3 minutos, que prova a arquitetura multiagente governada sem infraestrutura desnecessária.**

---

## 0. Decisões-chave (resumo)

| # | Decisão | Por quê |
|---|---|---|
| D1 | **Um único backend (FastAPI) em processo único.** Orquestrador, agentes, governança, tools e dados vivem no mesmo processo. | Zero microserviços, zero fila, zero rede entre agentes. Menos pontos de falha na demo. |
| D2 | **Agentes se comunicam apenas via Orquestrador, trocando JSON validado (Pydantic).** Nenhum agente chama outro agente. | Espelha o README (§9, §20): contexto mínimo, outputs compactos, auditáveis. |
| D3 | **Governança é código determinístico, não prompt.** Toda tool passa por um `ToolGateway` que autoriza, audita e carimba `source_id`. | README §14, §16, §30, §31. O LLM não consegue contornar. |
| D4 | **Estado do caso em memória + snapshot JSON em disco.** Sem banco. | Um caso por demo; restart não perde o caso; simples de inspecionar. |
| D5 | **Frontend SPA (React) consome a API via polling de eventos.** Uma URL pública, backend serve o build estático. | Um único deploy, um único link sem login. |
| D6 | **Provider LLM abstraído + `MockProvider` determinístico.** `DEMO_MODE=true` garante o roteiro (inclusive a inconsistência planejada). | Demo nunca depende de o LLM "acertar". Cálculos são feitos em código. |
| D7 | **Human Gate é um estado da máquina de estados do caso**, não um botão cosmético. O caso não termina sem `POST /human-review`. | README §18, §52. |

---

## 1. Visão geral

```text
┌──────────────────────────────────────────────────────────────────────────┐
│  FRONTEND (React SPA)                                                    │
│  Tela 1 Demanda → Tela 2 Squad em execução → Tela 3 Replanejamento →     │
│  Tela 4 Resultado + Human Gate                                           │
│  (polling GET /api/cases/{id}/events?after=N a cada 1s)                  │
└───────────────▲──────────────────────────────────────────────────────────┘
                │ HTTP/JSON
┌───────────────┴──────────────────────────────────────────────────────────┐
│  BACKEND (FastAPI, processo único)                                       │
│                                                                          │
│  api/            REST: cases, run, events, input, human-review, registry │
│                                                                          │
│  orchestration/  Orchestrator ─ Planner ─ Executor (asyncio)             │
│                    │ seleciona agentes no Registry, monta contexto       │
│                    │ mínimo, executa, roda Review, replaneja, consolida  │
│                    ▼                                                     │
│  agents/         Eligibility │ CreditRisk │ Structuring │ Review         │
│                    │ cada um: playbook + schema de output + tools        │
│                    ▼                                                     │
│  governance/     IAM (users) ─ PolicyEngine (authorize) ─ AuditLog       │
│                    │                                                     │
│  tools/          ToolGateway → tools mock (financials, agro, market,     │
│                    products, policy search, calculations)                │
│                    ▼                                                     │
│  knowledge/      Retriever (BM25/keyword) sobre docs mock → source_ids   │
│  data/           JSON mock (clients, financials, agro, market, products) │
│                                                                          │
│  services/       LLMProvider (OpenAI-compat | Mock) ─ Telemetry          │
│  store/          CaseStore (memória + snapshot JSON) ─ EventLog          │
└──────────────────────────────────────────────────────────────────────────┘
```

Mapeamento para as cinco camadas do README §7:

| Camada README | Módulo(s) |
|---|---|
| Orchestration | `orchestration/` |
| Agent | `agents/` + `registry/agents.json` |
| Data / Knowledge | `tools/`, `knowledge/`, `data/` |
| Governance | `governance/` (+ `ToolGateway`) |
| Human | estado `human_review_required` + endpoint `/human-review` + componente `HumanGate` |
| Transversal (quality/observability) | `schemas/` (validação), `services/telemetry.py`, `store/event_log.py`, `tests/` |

---

## 2. Componentes

### 2.1 API (`backend/app/api/`)

| Endpoint | Função |
|---|---|
| `POST /api/cases` | Cria caso a partir de `{user_id, prompt}`. Emite `CASE_CREATED`. |
| `POST /api/cases/{id}/run` | Dispara a execução em background (`asyncio.create_task`). Retorna 202. |
| `GET /api/cases/{id}` | Estado consolidado do caso (`CaseState`). |
| `GET /api/cases/{id}/events?after=N` | Eventos com `seq > N`. Base da timeline. |
| `POST /api/cases/{id}/input` | Responde `MISSING_INFO_REQUESTED`; retoma a execução. |
| `POST /api/cases/{id}/human-review` | `{decision: approve_next_step \| request_adjustment, comment}`. |
| `GET /api/registry/agents` | Agent Cards (para o painel visual do registry). |
| `GET /api/cases/{id}/audit` | Audit events (visão de governança). |
| `GET /api/health` | Liveness + `demo_mode` + provider ativo. |

A API é fina: valida entrada, delega ao `CaseService`, nunca contém lógica de negócio.

### 2.2 Orquestrador (`backend/app/orchestration/`)

Três peças, todas em código Python (o LLM é usado apenas onde há linguagem natural):

- **`planner.py`** — `plan(case) -> ExecutionPlan`
  1. Classifica intent e extrai campos (`client`, `amount`, `purpose`, `crop`, `cycle`) — LLM pequeno em `intent_extraction`, com fallback regex/heurística.
  2. Verifica `missing_information` contra os campos obrigatórios do intent.
  3. Seleciona agentes por *capability match* no Registry (`credito_agro` → `eligibility`, `credit_risk`, `structuring`, `review`).
  4. Define dependências: `eligibility ∥ credit_risk` → `structuring` (depende de risk) → `review`.
- **`executor.py`** — executa o plano em waves com `asyncio.gather`, monta o **contexto mínimo** de cada agente (`context_builder.py`), valida o output contra o schema, faz o *grounding check* (ver §6), registra telemetria e emite eventos.
- **`orchestrator.py`** — máquina de estados do caso (ver §3), loop de review (máx. 2 reexecuções), consolidação do `FinalResult`, Human Gate.

O Orquestrador **não** acessa tools de dados diretamente, não calcula, não aprova — só coordena (README §9).

### 2.3 Agentes (`backend/app/agents/`)

Todos herdam de `BaseAgent`:

```python
class BaseAgent:
    card: AgentCard                 # carregado de registry/agents.json
    output_schema: type[BaseModel]  # contrato de saída
    playbook: str                   # texto do playbook (knowledge/playbooks/*.md)

    async def run(self, task: AgentTask, ctx: ExecutionContext) -> AgentOutput:
        # 1. chama tools via ctx.tools.call(...)  (passa por governança)
        # 2. cálculos determinísticos em código
        # 3. LLM apenas para interpretação/narrativa, com schema JSON forçado
        # 4. retorna output validado + evidence_ids
```

| Agente | Pergunta | Tools permitidas | Determinístico em código | LLM |
|---|---|---|---|---|
| `agro_eligibility` | "Está completo e enquadrável?" | `get_client_profile`, `get_available_documents`, `get_agro_profile`, `search_policy` | checklist de documentos, comparação de área entre fontes (gera `DOC_AREA_MISMATCH`) | redigir warnings/pendências |
| `agro_credit_risk` | "Consegue pagar? Quais riscos?" | `get_client_financials`, `get_agro_profile`, `get_market_data`, `calculate_credit_metrics`, `run_stress_scenarios`, `search_policy` | métricas (dívida líq./EBITDA, geração de caixa), stress (preço −15%, produtividade −10%) | `risk_summary`, riscos e mitigantes em linguagem |
| `agro_structuring` | "Como estruturar?" | `get_product_catalog`, `search_policy`, **output do risk** (via contexto) | templates de alternativas A/B a partir do catálogo + prazo do ciclo | `rationale` e trade-offs |
| `credit_review` | "O que está errado ou faltando?" | nenhuma tool de dados; recebe apenas os outputs | **regras determinísticas**: produtividade assumida > histórico sem justificativa, `evidence_ids` inválidos, estrutura ignora restrição, campos obrigatórios ausentes, nenhum agente "aprova" | crítica adicional (issues extras, severidade ≤ medium) |

Regra de ouro: **qualquer número que apareça na UI vem de código, não do LLM.**

### 2.4 Agent Registry (`backend/app/registry/agents.json`)

Fonte de verdade dos Agent Cards (formato do README §13). O Planner só seleciona agentes que existem aqui. Cada card define `capabilities`, `tools`, `allowed_data_domains`, `forbidden_actions`, `human_gate_required_for`. Exposto em `GET /api/registry/agents` para o painel visual (P1).

### 2.5 Governança (`backend/app/governance/`)

Ver §4.

### 2.6 Tools e dados (`backend/app/tools/`, `backend/data/`)

Funções Python puras sobre JSON mock. Todas registradas num `TOOL_REGISTRY` com metadados `{name, data_domain, source_id_prefix}`. Só são invocáveis via `ToolGateway`.

### 2.7 Knowledge (`backend/app/knowledge/`)

Documentos curtos em Markdown (`POL-AGRO-001`, `POL-CRED-002`, `PLAYBOOK-RISK-001`, `CATALOG-AGRO-001`, `GLOSSARY-001`). `retriever.search(query, k)` faz ranking BM25/keyword sobre chunks e devolve `[{source_id: "POL-CRED-002#2", text, score}]`. Sem vector DB.

### 2.8 Store e eventos (`backend/app/store/`)

`CaseStore`: `dict[case_id, Case]` + `save(case)` que grava `runtime/cases/{case_id}.json`. `EventLog`: lista append-only por caso com `seq` monotônico.

### 2.9 Frontend (`frontend/src/`)

| Componente | Tela | Fonte de dados |
|---|---|---|
| `CaseInput` | 1 — Demanda | `POST /cases` + `/run` |
| `SquadTimeline`, `AgentCard` | 2 — Squad em execução | eventos `AGENT_*`, `TOOL_CALLED`, `PERMISSION_CHECKED` |
| `ReplanBanner` | 3 — Replanejamento | eventos `REVIEW_ISSUE_FOUND`, `TASK_REOPENED` |
| `ResultPanel`, `EvidencePanel`, `GovernancePanel` | 4 — Resultado | `GET /cases/{id}` (`final_result`, `sources`, `audit`) |
| `HumanGate` | 4 — Decisão | `POST /human-review` |
| `MetricsBar` | transversal | `case.metrics` |

---

## 3. Como um caso executa (máquina de estados)

```text
created ──run──▶ planning ──faltou info──▶ waiting_input ──/input──▶ planning
                    │
                    ▼ plano ok
              executing_agents  (wave 1: eligibility ∥ credit_risk ; wave 2: structuring)
                    │
                    ▼
                reviewing ──issues materiais & loops<2──▶ reworking ──▶ reviewing
                    │
                    ▼ review passed (ou loops esgotados → issues viram pendências)
               consolidating
                    │
                    ▼
          human_review_required ──request_adjustment──▶ reworking ──▶ reviewing ──▶ ...
                    │
                    ▼ approve_next_step
                completed
```

Sequência do caso demo (`DEMO_MODE=true`), com os eventos emitidos:

```text
CASE_CREATED
ORCHESTRATOR_STARTED
AGENT_SELECTED ×4                        (eligibility, credit_risk, structuring, review)
PERMISSION_CHECKED / TOOL_CALLED ×N      (um par por tool call, com allowed=true|false)
AGENT_STARTED eligibility ∥ AGENT_STARTED credit_risk
AGENT_COMPLETED eligibility              (ready_with_warnings, 1 alerta DOC_AREA_MISMATCH)
AGENT_COMPLETED credit_risk v1           (usa produtividade 61)
AGENT_STARTED / AGENT_COMPLETED structuring v1
REVIEW_STARTED
REVIEW_ISSUE_FOUND                       (PRODUCTIVITY_ASSUMPTION, high, owner=credit_risk)
TASK_REOPENED credit_risk                (constraint: productivity_baseline=historical)
AGENT_STARTED / AGENT_COMPLETED credit_risk v2   (stress recalculado com 58)
TASK_REOPENED structuring                (depende de risk → recebe v2)
AGENT_COMPLETED structuring v2
REVIEW_STARTED → review passed
RESULT_CONSOLIDATED
HUMAN_REVIEW_REQUIRED
HUMAN_APPROVED | HUMAN_ADJUSTMENT_REQUESTED
CASE_COMPLETED
```

Guardas: máximo de **2 loops de review** e **1 loop de ajuste humano por rodada**; se o limite estourar, os issues restantes viram `pending_items` no resultado e o caso vai para o Human Gate mesmo assim (o humano decide).

---

## 4. Como os agentes se comunicam

**Não há comunicação direta agente↔agente.** Todo fluxo passa pelo Orquestrador, em processo, via objetos Pydantic:

```text
Orchestrator ──AgentTask──▶ Agent ──AgentOutput──▶ Orchestrator ──(slice)──▶ próximo Agent
```

- **`AgentTask`**: `{task_id, case_id, agent_id, objective, inputs, constraints, allowed_tools, context_refs}`.
  - `inputs`: só os campos que o agente precisa (ex.: structuring recebe `requested_amount`, `cycle`, `risk.repayment_capacity`, `risk.main_risks`, `eligibility.warnings` — **não** recebe os financials brutos).
  - `constraints`: injetadas em reexecução (ex.: `{"productivity_baseline": "historical", "reason": "REVIEW:PRODUCTIVITY_ASSUMPTION"}`).
  - `context_refs`: `source_ids` de outputs anteriores (ex.: `SRC_RISK_OUTPUT_v2`) — o agente referencia, não copia.
- **`AgentOutput`**: `{agent_id, version, payload: <schema específico>, evidence_ids, metrics: {latency_ms, tokens_in, tokens_out, tool_calls, model}}`.
- **`context_builder.build(agent_id, case) -> dict`** é a única função que decide o que cada agente vê. É aqui que se materializa "contexto mínimo" (README §21, §25). Fácil de testar e de mostrar na demo ("Structuring recebeu 6 campos, não 40").
- **Eventos** são o canal de comunicação com o frontend e o audit trail, nunca o canal entre agentes.

Por que não uma fila/bus: um caso por vez, quatro agentes, execução < 60s. `asyncio.gather` resolve paralelismo; um bus só adicionaria latência e pontos de falha.

---

## 5. Como simulamos permissões

### 5.1 Três fontes de verdade (todas JSON, deterministas)

```text
governance/users.json        USER-DEMO-001: permissions ["client_profile:read", "client_financials:read",
                                              "agro_profile:read", "market_mock:read", "credit_products:read",
                                              "policies:read", "documents:read"]
                             USER-DEMO-002 (P2): sem "client_financials:read" → demonstra bloqueio por usuário

registry/agents.json         agro_structuring.allowed_data_domains = ["credit_products", "policies"]
                             (NÃO inclui client_financials → demonstra bloqueio por agente)

governance/purposes.json     "credito_agro_analysis": ["client_profile","client_financials","agro_profile",
                                                       "market_mock","credit_products","policies","documents"]
```

### 5.2 Policy Engine

```python
def authorize(user: UserContext, agent: AgentCard, resource: str, purpose: str) -> Decision:
    if f"{resource}:read" not in user.permissions:      return Decision(False, "resource_not_authorized_for_user")
    if resource not in agent.allowed_data_domains:      return Decision(False, "resource_not_authorized_for_agent")
    if resource not in PURPOSE_RESOURCES[purpose]:      return Decision(False, "resource_not_authorized_for_purpose")
    return Decision(True, "ok")
```

Acesso efetivo = usuário ∩ agente ∩ propósito (README §14). Não existe caminho para o LLM alterar isso: agentes nunca recebem as funções de tool, só o `ToolGateway`.

### 5.3 ToolGateway (único ponto de acesso a dados)

```python
async def call(self, tool_name: str, **args) -> ToolResult:
    tool = TOOL_REGISTRY[tool_name]
    decision = authorize(self.ctx.user, self.ctx.agent, tool.data_domain, self.ctx.purpose)
    audit.record(ctx=self.ctx, action=tool_name, resource=args.get("client_id", "-"), allowed=decision.allowed, reason=decision.reason)
    events.emit("PERMISSION_CHECKED", {...decision})
    if not decision.allowed:
        return ToolResult(ok=False, reason=decision.reason)     # o agente segue sem o dado
    data = tool.fn(**args)
    result = ToolResult(ok=True, data=data, source_id=tool.source_id(args))
    self.ctx.sources_seen.add(result.source_id)                  # usado no grounding check
    events.emit("TOOL_CALLED", {...})
    return result
```

Consequências que a demo mostra:

- **No privilege union** (README §16): a autorização é por tool call com o card do agente que está executando; combinar agentes nunca soma domínios. Teste: `credit_risk` pode `client_financials`, `structuring` não; a squad inteira **não** consegue via structuring.
- **Bloqueio visível**: no roteiro demo, o Structuring Agent tenta `get_client_financials` (para "checar caixa") e recebe `resource_not_authorized_for_agent`. O GovernancePanel mostra "Structuring Agent tentou acessar dados financeiros — **bloqueado**. Usou o output do Risk Agent." Isso vale mais que dez slides.
- **Identity propagation** (README §15): `ExecutionContext = {user, agent, case_id, task_id, purpose}` é criado pelo Executor por task e injetado no gateway; nenhuma tool é chamável sem ele.

### 5.4 Ações proibidas

`forbidden_actions` (`approve_credit`, `send_client_proposal`) não existem como tools. Adicionalmente, o schema do `FinalResult` não tem campo "aprovado" — só `status: "ready_for_human_review"` e `recommendation`. Um teste garante que nenhum output de agente contém `approved: true`.

---

## 6. Como armazenamos contexto

### 6.1 Um objeto por caso

```python
class Case(BaseModel):
    case_id: str
    user_id: str
    prompt: str
    status: CaseStatus
    intent: str | None
    extracted: dict            # campos extraídos da demanda
    missing_information: list[str]
    plan: ExecutionPlan | None
    agent_outputs: dict[str, list[AgentOutput]]   # agent_id -> versões (v1, v2...) — rework não apaga histórico
    reviews: list[ReviewOutput]
    final_result: FinalResult | None
    human_decisions: list[HumanDecision]
    sources: dict[str, SourceRef]                 # source_id -> metadados (ver §7)
    audit: list[AuditEvent]
    metrics: CaseMetrics                          # agregados de telemetria
    events: list[Event]                           # append-only, seq monotônico
```

Vive em `CaseStore` (dict em memória). Após cada transição de estado é serializado em `runtime/cases/{case_id}.json` — sobrevive a restart e é um artefato legível para debug e para o vídeo.

### 6.2 As quatro camadas de conhecimento (README §25) e onde moram

| Nível | Conteúdo | Onde | Quem acessa |
|---|---|---|---|
| 1 Global | glossário, ontologia | `knowledge/docs/GLOSSARY-001.md` | qualquer agente via `search_policy` |
| 2 Área | políticas, playbooks, catálogo | `knowledge/docs/POL-*.md`, `PLAYBOOK-*.md`, `CATALOG-*.md` | agentes autorizados a `policies` / `credit_products` |
| 3 Caso | dados do cliente, mercado, docs | `data/*.json` via tools | só agentes com domínio autorizado |
| 4 Temporário | descobertas desta squad | `case.agent_outputs`, `case.reviews` | Orquestrador decide o *slice* via `context_builder` |

Playbooks entram no *system prompt* do agente (nível 2, curtos). Dados de caso **nunca** entram no prompt do Orquestrador — só nos agentes que os buscaram.

### 6.3 O que o LLM vê

Cada chamada de LLM recebe: playbook do agente (~300 tokens) + `AgentTask.inputs` compacto + resultados das tools já resumidos em código + instrução de schema JSON. Nenhum agente recebe a conversa inteira ou documentos brutos.

---

## 7. Como rastreamos fontes

### 7.1 Todo dado tem `source_id`

| Origem | Formato | Exemplo |
|---|---|---|
| Tool de dados | `SRC_<DOMÍNIO>` (README) + `SourceRef` detalhado | `SRC_FINANCIALS` → `{tool: get_client_financials, resource: CLIENTE-001, retrieved_at, fields: [...]}` |
| Knowledge (RAG) | `<DOC-ID>#<chunk>` | `POL-CRED-002#1` → `{title: "Regra mock de alavancagem", excerpt: "..."}` |
| Cálculo | `SRC_CALC_<NOME>` | `SRC_CALC_STRESS_v2` → `{inputs: {...}, formula: "..."}` |
| Output de outro agente | `SRC_<AGENT>_OUTPUT_v<n>` | `SRC_RISK_OUTPUT_v2` |

O `ToolGateway` registra cada `SourceRef` em `case.sources` no momento da chamada.

### 7.2 Grounding check (determinístico)

Ao receber um `AgentOutput`, o Executor valida: `evidence_ids ⊆ ctx.sources_seen`. IDs desconhecidos são removidos e geram `warning GROUNDING_UNKNOWN_SOURCE`; o Review Agent lê esses warnings e marca `grounding_ok=false` se sobrar algo material. Isso implementa "não inventar fontes" (README §9) sem confiar no LLM.

### 7.3 Na UI

- **Source chips** em cada card de agente (`SRC_FINANCIALS`, `POL-CRED-002#1`).
- **EvidencePanel** no resultado: lista `case.sources` com tipo, título, quem acessou, quando; clique abre o excerpt/campos.
- **Audit timeline**: "Risk Agent acessou dados financeiros — autorizado" (README §32).

---

## 8. LLM, determinismo e Modo Demo

### 8.1 Provider abstrato

```python
class LLMProvider(Protocol):
    async def complete_json(self, *, system: str, user: str, schema: type[BaseModel], model_tier: Literal["small","strong"]) -> tuple[BaseModel, Usage]
```

Implementações: `OpenAICompatProvider` (OpenAI, Azure, OpenRouter, Groq, Ollama — troca por `LLM_BASE_URL`/`LLM_MODEL_*`) e `MockProvider` (respostas fixas por `agent_id` + `version`). `model_tier` permite roteamento barato/forte (README §22) via env, sem tocar código.

### 8.2 Onde o LLM entra e onde não entra

| Passo | Quem faz |
|---|---|
| Extração de intent/campos da demanda | LLM small (fallback: regex) |
| Seleção de agentes, dependências | código (registry) |
| Autorização, audit, source_ids | código |
| Métricas, stress, checklist de documentos, comparação de área | código |
| Textos: warnings, riscos, mitigantes, rationale, resumo executivo | LLM strong com schema forçado |
| Regras do Review (produtividade, grounding, restrições) | código |
| Crítica adicional do Review | LLM strong (issues extras nunca bloqueiam sozinhos: severidade ≤ medium) |
| Consolidação do FinalResult | código + template (LLM só para o parágrafo de resumo) |

### 8.3 `DEMO_MODE=true`

- Case pré-carregado (`CASE-AGRO-001`, Fazenda Horizonte S.A., R$ 50 mi, soja 2026/27).
- Tools 100% determinísticas.
- Inconsistência planejada garantida por **regra de código**: `agro_profile.expected_productivity (61) > historical_productivity (58) × 1,03` e sem `justification` → o Review emite `PRODUCTIVITY_ASSUMPTION` na rodada 1. Na rodada 2, a constraint `productivity_baseline=historical` faz o Risk usar 58; o stress `productivity_minus_10pct` muda de `still_payable` para `attention_required`, e a estrutura preferida ganha a condicionante "seguro agrícola / trava de preço".
- Bloqueio planejado: Structuring tenta `client_financials` → negado.
- **Fallback**: se o provider real falhar ou exceder `LLM_TIMEOUT_S`, o agente usa o `MockProvider` para aquela chamada e emite `AGENT_FALLBACK_USED` (fica visível, não escondido). A demo nunca quebra por causa de LLM.
- `DEMO_MODE=false` mantém tudo igual, mas sem fallback silencioso e com usuário/permissões vindos do header `X-User-Id`.

---

## 9. Como um usuário percorre a jornada

| Passo | Usuário vê | O que acontece |
|---|---|---|
| 1 | **Tela 1 – Demanda**: campo com exemplo pré-preenchido, banner "Ambiente demonstrativo — dados fictícios", botão **Montar squad** | `POST /cases` → `POST /run`; frontend começa polling de `/events` |
| 2 | **Tela 2 – Squad**: card do Orquestrador ("Entendeu demanda", "Selecionou 4 agentes"); 4 cards de agentes com barra de progresso e badge de status; linha "Dados acessados respeitando permissões de USER-DEMO-001" | `ORCHESTRATOR_STARTED`, `AGENT_SELECTED`, `AGENT_STARTED`… Cards enchem conforme `TOOL_CALLED`/`PERMISSION_CHECKED`; um chip vermelho "bloqueado" aparece no Structuring |
| 3 | **Tela 3 – Replanejamento** (banner sobre a Tela 2): "⚠ Produtividade assumida está 5,2% acima do histórico sem justificativa" → seta "Review → Orquestrador → Reabrindo tarefa do Agro Risk Agent → Stress recalculado" | `REVIEW_ISSUE_FOUND`, `TASK_REOPENED`, Risk card volta a "trabalhando" e vira v2, Structuring v2 |
| 4 | **Tela 4 – Resultado**: Resumo, Capacidade de pagamento (métricas + stress v1 vs v2), Riscos/mitigantes, Estruturas A/B, Pendências, Evidências (source chips), Governança (usuário, agentes, permissões, bloqueios), Métricas (tempo, agentes, tool calls, tokens, fontes, loops) | `RESULT_CONSOLIDATED`, `HUMAN_REVIEW_REQUIRED`; `GET /cases/{id}` |
| 5 | **Human Gate**: "Decisões materiais permanecem sob responsabilidade humana" + `[Solicitar ajuste]` `[Aprovar para próxima etapa]` | `request_adjustment` (com comentário) → reabre Structuring e re-review → volta ao Human Gate; `approve_next_step` → `CASE_COMPLETED` e tela final "Análise gerada para suporte à decisão. Não representa aprovação de crédito." |

Tempo-alvo do fluxo completo: **< 60 s** com LLM real, **< 10 s** com `MockProvider` (usando `DEMO_STEP_DELAY_MS` para dar ritmo visual à timeline).

---

## 10. Menor MVP demonstrável (walking skeleton)

O menor artefato que já conta a história inteira — e que deve existir **antes** de qualquer chamada real a LLM:

1. Backend com `MockProvider`, 4 agentes com outputs fixos, ToolGateway real (autorização + audit + source_id), Review com regra de produtividade, loop de rework, Human Gate.
2. Frontend com Telas 1→4 dirigidas exclusivamente pelos eventos.
3. Um único caso (`CASE-AGRO-001`), um único usuário (`USER-DEMO-001`).
4. Deploy em uma URL pública.

Isso já cumpre 18 dos 20 critérios de aceite do README §53 (faltam apenas "orquestrador interpreta" com LLM e textos gerados). A partir daí, cada incremento é substituível sem quebrar a demo:

| Incremento | Substitui | Risco se faltar |
|---|---|---|
| LLM real para textos dos agentes | strings fixas do Mock | baixo — demo continua funcionando |
| LLM small para extração de intent + fluxo `waiting_input` | regex | baixo |
| Retriever BM25 sobre docs mock | `search_policy` retornando chunk fixo | baixo |
| Métricas de tokens/latência na UI | placeholder | baixo |
| Registry visual, 2º usuário com menos permissões | — | nenhum (P1/P2) |

---

## 11. Estrutura do repositório

```text
/
├─ README.md  ARCHITECTURE.md  TECH_STACK.md  TASKS.md
├─ .env.example
├─ Dockerfile                      # multi-stage: build frontend → copia para backend/static
├─ backend/
│  ├─ app/
│  │  ├─ main.py                   # FastAPI app, monta /api e serve /static (SPA)
│  │  ├─ config.py                 # Settings (pydantic-settings): DEMO_MODE, LLM_*, ...
│  │  ├─ api/            cases.py  registry.py  audit.py  health.py
│  │  ├─ orchestration/  orchestrator.py  planner.py  executor.py  context_builder.py  consolidator.py
│  │  ├─ agents/         base.py  eligibility.py  credit_risk.py  structuring.py  review.py
│  │  ├─ registry/       agents.json  loader.py
│  │  ├─ governance/     users.json  purposes.json  iam.py  policy_engine.py  audit.py
│  │  ├─ tools/          gateway.py  registry.py  financials.py  agro.py  market.py  products.py  knowledge.py  calculations.py
│  │  ├─ knowledge/      retriever.py  docs/  playbooks/
│  │  ├─ schemas/        case.py  tasks.py  outputs.py  events.py  audit.py  registry.py
│  │  ├─ services/       llm.py  llm_mock.py  telemetry.py
│  │  └─ store/          case_store.py  event_log.py
│  ├─ data/              clients.json  financials.json  agro_profiles.json  market_data.json  products.json  documents.json  historical_cases.json
│  ├─ runtime/cases/     (gitignored; snapshots)
│  └─ tests/             test_policy_engine.py  test_gateway.py  test_agents_schema.py  test_orchestrator_loop.py  test_demo_case.py
├─ frontend/
│  ├─ src/
│  │  ├─ App.tsx  main.tsx
│  │  ├─ lib/          api.ts  types.ts (espelho dos schemas)  useCaseEvents.ts  reducer.ts
│  │  ├─ components/   CaseInput  SquadTimeline  AgentCard  ReplanBanner  ResultPanel  EvidencePanel  GovernancePanel  HumanGate  MetricsBar  DemoBanner
│  │  └─ mocks/        events.demo.json  (permite desenvolver a UI sem backend)
│  └─ package.json  vite.config.ts  tailwind.config.js
└─ docs/
   ├─ api.md            # contratos (gerado do OpenAPI + eventos)
   ├─ demo-script.md
   ├─ agent-cards.md
   └─ submission-checklist.md
```

---

## 12. Riscos e mitigações

| Risco | Mitigação |
|---|---|
| LLM lento/indisponível no dia | `MockProvider` + fallback automático por chamada; `DEMO_MODE` |
| LLM não "encontra" a inconsistência | Review tem regras em código; o LLM só adiciona |
| Números incoerentes entre agentes | Todo número vem de `calculations.py`; agentes só referenciam |
| Frontend bloqueado esperando backend | Contratos + `mocks/events.demo.json` no dia 1; stub de API que reproduz a sequência |
| Deploy quebra na véspera | Um Dockerfile, um serviço; deploy contínuo desde o dia 1; fallback Vercel+Render |
| Restart perde o caso na apresentação | Snapshot JSON; case demo é recriável em 1 clique |
| Parecer "quatro chatbots" | UI = central de execução: cards, timeline, chips, badges; zero bolhas de chat |

---

## 13. Fora da arquitetura (deliberadamente)

Microserviços, filas, Kubernetes, banco relacional, vector DB, autenticação real, multi-tenant, streaming token-a-token, frameworks de agentes pesados (LangGraph/CrewAI/AutoGen). Todos aumentariam risco e tempo sem melhorar a história que a demo conta. O `README.md` §56–57 descreve como a mesma arquitetura escala depois: novos Agent Cards no Registry, novos domínios em `purposes.json`, novas tools no `TOOL_REGISTRY` — o core não muda.
