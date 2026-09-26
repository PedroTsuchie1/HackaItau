# TASKS.md — Plano de trabalho para 3 devs em paralelo

> Referências: `ARCHITECTURE.md` (o quê e por quê), `TECH_STACK.md` (com o quê). Este documento diz **quem faz o quê, em que ordem, e como as partes se encontram**.
> Premissa: ~2 dias de hackathon (≈ 20–28 h de trabalho por pessoa). Ajuste os horários, não a ordem.

---

## 0. Divisão de papéis

| Dev | Papel | Dono de | Não toca em |
|---|---|---|---|
| **Dev A — Plataforma & Governança** | backend "de baixo para cima": contratos, store, eventos, API, governança, tools, dados mock, knowledge, telemetria, Docker/deploy, CI | `backend/app/{api,governance,tools,knowledge,registry,schemas,store,services/telemetry.py,config.py,main.py}`, `backend/data/`, `Dockerfile`, `.github/` | `orchestration/`, `agents/`, `frontend/` |
| **Dev B — Orquestração & Agentes** | o "cérebro": provider LLM, 4 agentes, planner/executor/orchestrator, review loop, human gate, determinismo da demo, testes de fluxo | `backend/app/{orchestration,agents,services/llm*.py}`, `knowledge/playbooks/`, `tests/test_orchestrator*.py`, `tests/test_agents*.py`, `tests/test_demo_case.py` | `frontend/`, `governance/`, `tools/` (usa via gateway) |
| **Dev C — Frontend & Demo** | tudo que o jurado vê: Telas 1–4, timeline, painéis, human gate, disclaimers, roteiro da demo, apoio a slides/vídeo | `frontend/`, `docs/demo-script.md`, `docs/submission-checklist.md` | `backend/` (exceto `mocks` e `types.ts`) |

**Regra de integração**: os contratos em `backend/app/schemas/` e `docs/api.md` são a fronteira. Mudança de contrato = avisar os outros dois **antes** de commitar.

---

## 1. Linha do tempo

```text
H0  ─┬─ Fase 0  KICKOFF (todos juntos, ~1,5 h)       contratos fechados
     │
H2  ─┼─ Fase 1  WALKING SKELETON (~6 h)               ponta a ponta com MockProvider
     │            checkpoint 1: demo roda em localhost sem LLM
H8  ─┼─ Fase 2  FEATURES REAIS (~10 h)                LLM real, RAG, métricas, UI completa
     │            checkpoint 2: demo roda com LLM + fallback, deploy público funcionando
H18 ─┼─ Fase 3  POLIMENTO & ENTREGA (~6 h)            ensaio, vídeo, slides, ficha, congelamento
     │            checkpoint 3: 4 entregas na mesma versão, links testados em aba anônima
H24 ─┴─ FREEZE
```

Checkpoints são reuniões de 15 min: cada um demonstra sua parte rodando, não descreve.

---

## 2. Fase 0 — Kickoff (todos, ~1,5 h)

Objetivo: **nenhum dev fica esperando outro**. Saídas obrigatórias, todas commitadas em `main` ao fim da fase:

| # | Tarefa | Dono | Saída |
|---|---|---|---|
| 0.1 | Criar esqueleto do repo (pastas de `ARCHITECTURE.md §11`), `.env.example`, `.gitignore`, `Makefile`, `pyproject`/`requirements.txt`, `package.json` | A | repo roda `make dev` (vazio) |
| 0.2 | Escrever **schemas** Pydantic: `Case`, `CaseStatus`, `ExecutionPlan`, `AgentTask`, `AgentOutput`, `EligibilityOutput`, `RiskOutput`, `StructuringOutput`, `ReviewOutput`, `FinalResult`, `Event`, `EventType`, `AuditEvent`, `SourceRef`, `AgentCard`, `HumanDecision`, `CaseMetrics` | A + B (pair) | `backend/app/schemas/*.py` |
| 0.3 | Fixar **event model** (README §50 + `AGENT_FALLBACK_USED`, `HUMAN_ADJUSTMENT_REQUESTED`) e payload de cada evento | B | `docs/api.md` §Eventos |
| 0.4 | Fixar **API** (endpoints, request/response) | A | `docs/api.md` §Endpoints |
| 0.5 | Gerar `frontend/src/lib/types.ts` espelhando schemas e eventos | C | `types.ts` |
| 0.6 | Escrever à mão `frontend/src/mocks/events.demo.json`: a sequência completa do caso demo (§3 de `ARCHITECTURE.md`), ~40 eventos, com payloads realistas | C (revisão B) | UI pode ser construída 100% offline |
| 0.7 | Fixar **roteiro determinístico da demo**: valores mock (financials, agro profile 61/58, market 135), inconsistência planejada, bloqueio planejado (Structuring → financials), resultado v1 vs v2 | B (revisão A) | `docs/demo-script.md` v0 |
| 0.8 | Definir paleta/tipografia e wireframe rápido das 4 telas (papel/Excalidraw) | C | imagem em `docs/` |

---

## 3. Fase 1 — Walking skeleton (H2 → H8)

Meta: **checkpoint 1** = `make demo` roda o caso inteiro com `MockProvider`, a UI mostra as 4 telas alimentadas pelo backend real, human gate encerra o caso.

### Dev A

| ID | Tarefa | DoD |
|---|---|---|
| A1 | `config.py` (Settings), `main.py` (app, CORS, `/api/health`, servir `static/` com fallback SPA) | `GET /api/health` → `{status, demo_mode, llm_provider}` |
| A2 | `store/case_store.py` + `store/event_log.py` (seq monotônico, `after=N`, snapshot JSON em `runtime/cases/`) | teste: 100 eventos, `after=50` retorna 50 |
| A3 | `api/cases.py`: `POST /cases`, `POST /cases/{id}/run` (dispara `orchestrator.run` de B via `asyncio.create_task`), `GET /cases/{id}`, `GET /cases/{id}/events`, `POST /cases/{id}/input`, `POST /cases/{id}/human-review` | todos respondem com schemas; `run` em caso já rodando → 409 |
| A4 | **Stub temporário** `orchestration/_stub_runner.py` que reproduz `events.demo.json` com delay, para C integrar enquanto B constrói o orquestrador real | C consegue ver a timeline com backend real em H4 |
| A5 | Dados mock `backend/data/*.json` com `_meta.mock=true` (cliente, financials, agro, market, products, documents, historical_cases) conforme README §26–28 | JSON válidos; valores idênticos a `demo-script.md` |
| A6 | `governance/users.json`, `purposes.json`, `iam.py` (`get_user_context`), `policy_engine.py` (`authorize`) | `tests/test_policy_engine.py`: nega por usuário, por agente, por purpose; **no privilege union** |
| A7 | `governance/audit.py` (`record` → `case.audit` + evento `PERMISSION_CHECKED`) | audit event tem todos os campos do README §32 |
| A8 | `tools/registry.py` (`TOOL_REGISTRY` com `data_domain`, `source_id`) + tools mock: `get_client_profile`, `get_client_financials`, `get_agro_profile`, `get_market_data`, `get_available_documents`, `get_product_catalog`, `get_historical_cases` | cada tool retorna `ToolResult{data, source_id}` |
| A9 | `tools/gateway.py` (`ToolGateway.call`: authorize → audit → evento → executa → registra `SourceRef` em `case.sources` e `ctx.sources_seen`) | `tests/test_gateway.py`: chamada negada não executa a tool, gera audit e `PERMISSION_CHECKED allowed=false` |
| A10 | `tools/calculations.py`: `calculate_credit_metrics`, `run_stress_scenarios(productivity_baseline=...)` | testes com os números do roteiro: com 61 → `still_payable`; com 58 → `attention_required` no cenário produtividade −10% |
| A11 | `registry/agents.json` (4 Agent Cards, README §13) + `loader.py` | `GET /api/registry/agents` |
| A12 | `Dockerfile`, `docker-compose.yml`, primeiro deploy no Render apontando para `main` | URL pública responde `/api/health` em H8 |

### Dev B

| ID | Tarefa | DoD |
|---|---|---|
| B1 | `services/llm.py` (`LLMProvider` Protocol, `Usage`) + `services/llm_mock.py` (`MockProvider` indexado por `(agent_id, version)`) | `complete_json` retorna instância do schema + usage fake |
| B2 | `agents/base.py` (`BaseAgent.run`, carrega card + playbook, recebe `ToolGateway` e `LLMProvider` por injeção) | agente sem tool permitida em `card.tools` levanta erro claro |
| B3 | `agents/eligibility.py` — checklist de docs em código, comparação de área → `DOC_AREA_MISMATCH`, output `EligibilityOutput` | `evidence_ids` = `[SRC_CLIENT_PROFILE, SRC_AGRO_REGISTRY, ...]` reais do gateway |
| B4 | `agents/credit_risk.py` — chama tools financials/agro/market → `calculate_credit_metrics` → `run_stress_scenarios` respeitando `constraints.productivity_baseline` | v1 usa 61; v2 (constraint `historical`) usa 58 e muda o stress |
| B5 | `agents/structuring.py` — lê `inputs.risk` (slice), `get_product_catalog`; **tenta** `get_client_financials` (bloqueio planejado) e segue; gera ALT-A/ALT-B | output inclui `preferred_for_discussion`; evidence inclui `SRC_RISK_OUTPUT_v{n}` |
| B6 | `agents/review.py` — regras determinísticas: `PRODUCTIVITY_ASSUMPTION` (assumida > histórico×1,03 sem justificativa), `GROUNDING_UNKNOWN_SOURCE`, `STRUCTURE_IGNORES_CONSTRAINT`, `MISSING_REQUIRED_FIELD`, `FORBIDDEN_APPROVAL_LANGUAGE`; `reexecution_required` + `owner_agent` | `tests/test_review_rules.py`: rodada 1 acha issue high; rodada 2 passa |
| B7 | `orchestration/planner.py` — extração de campos (regex/heurística nesta fase), `missing_information`, seleção por capabilities no registry, waves de dependência | plano do caso demo = 4 agentes, 2 waves + review |
| B8 | `orchestration/context_builder.py` — slice por agente (`build(agent_id, case)`) | teste: structuring **não** recebe `financials` brutos; recebe ≤ 8 campos |
| B9 | `orchestration/executor.py` — cria `ExecutionContext` por task, `asyncio.gather` por wave, valida schema, grounding check, telemetria, eventos `AGENT_STARTED/COMPLETED`, versionamento `v1, v2…` | outputs em `case.agent_outputs[agent_id]` como lista |
| B10 | `orchestration/orchestrator.py` — máquina de estados (`ARCHITECTURE.md §3`), loop de review (máx. 2), `TASK_REOPENED` do owner + dependentes, `consolidator.py` → `FinalResult`, `HUMAN_REVIEW_REQUIRED`, tratamento de `human-review` (`approve_next_step` → `CASE_COMPLETED`; `request_adjustment` → reabre structuring + re-review) | `tests/test_demo_case.py`: fluxo completo com `MockProvider` termina em `human_review_required`; após approve → `completed`; caso **não** termina sem human gate |
| B11 | Substituir o stub A4 pelo orquestrador real (`POST /run` chama `orchestrator.run`) | checkpoint 1 |

### Dev C

| ID | Tarefa | DoD |
|---|---|---|
| C1 | Bootstrap Vite + React + TS + Tailwind + lucide; layout base (header com nome do produto, `DemoBanner` fixo "Ambiente demonstrativo — dados fictícios") | `npm run dev` abre a casca |
| C2 | `lib/api.ts` (fetch tipado dos 7 endpoints) + `lib/useCaseEvents.ts` (polling `after=seq`, 1 s, para em `CASE_COMPLETED`) + `lib/reducer.ts` (eventos → `UiState`: status por agente, progresso, chips de fonte, bloqueios, issues, loops) | `vitest`: aplicar `events.demo.json` termina em `human_review_required` com 4 agentes `done` |
| C3 | Flag `VITE_USE_MOCKS` que alimenta o reducer com `events.demo.json` em ritmo de 600 ms | UI inteira navegável sem backend |
| C4 | **Tela 1** `CaseInput`: textarea pré-preenchida com a demanda do README §35, botão **Montar squad**, usuário exibido (`USER-DEMO-001`) | chama `POST /cases` + `/run` e navega para Tela 2 |
| C5 | **Tela 2** `SquadTimeline` + `AgentCard` ×4 + card do Orquestrador: barra de progresso, badge de status (aguardando / trabalhando / concluído / reaberto), contador de tool calls, chips de `source_id`, chip vermelho "bloqueado" quando `PERMISSION_CHECKED allowed=false`; linha "Dados acessados respeitando permissões de USER-DEMO-001" | dirigida só por eventos |
| C6 | **Tela 3** `ReplanBanner`: aparece em `REVIEW_ISSUE_FOUND`, mostra mensagem do issue, owner, e a cadeia "Review → Orquestrador → Reabrindo tarefa do X → recalculado"; card do owner volta a "trabalhando" e mostra "v2" | visível ≥ 4 s na demo |
| C7 | **Tela 4** `ResultPanel` (Resumo, Capacidade de pagamento com stress v1 vs v2, Riscos/Mitigantes, Estruturas A/B, Pendências) a partir de `GET /cases/{id}` | disclaimer "Análise gerada para suporte à decisão. Não representa aprovação de crédito." |
| C8 | `HumanGate`: texto "Decisões materiais permanecem sob responsabilidade humana", botões **Solicitar ajuste** (abre campo de comentário) e **Aprovar para próxima etapa**; estados pós-decisão | `request_adjustment` volta para Tela 2/3 e depois ao gate; `approve` mostra tela final |
| C9 | Integrar com backend real (stub A4 em H4; orquestrador real em H8) | checkpoint 1 |

---

## 4. Fase 2 — Features reais (H8 → H18)

Meta: **checkpoint 2** = demo com LLM real (e fallback automático), RAG, métricas, painéis de evidência/governança, deploy público estável.

### Dev A

| ID | Tarefa | DoD |
|---|---|---|
| A13 | `knowledge/docs/*.md` (POL-AGRO-001, POL-CRED-002, PLAYBOOK-RISK-001, CATALOG-AGRO-001, GLOSSARY-001) + `knowledge/retriever.py` (BM25 por seção, `source_id = DOC#n`) + tool `search_policy` no gateway (domínio `policies`) | `search_policy("alavancagem")` → `POL-CRED-002#1` |
| A14 | `services/telemetry.py`: por task e agregados em `case.metrics` (tempo total, nº agentes, tool calls, tokens por agente, nº fontes, nº issues, nº loops, custo estimado por tabela de preço em env) | `GET /cases/{id}` traz `metrics` preenchido |
| A15 | `GET /cases/{id}/audit` + `GET /registry/agents` finalizados; `docs/api.md` regenerado do OpenAPI | C consome sem perguntas |
| A16 | `DEMO_STEP_DELAY_MS` aplicado no executor (via hook fornecido por B) para ritmo visual quando `MockProvider` | timeline legível em vídeo |
| A17 | 2º usuário `USER-DEMO-002` sem `client_financials:read` (P2, só se houver tempo) + header `X-User-Id` | Risk Agent bloqueado → pendência "sem acesso a financials" no resultado |
| A18 | CI GitHub Actions (ruff + pytest + `npm run build`), pre-commit | verde em `main` |
| A19 | Deploy contínuo no Render estável; `docs/deploy.md` com plano B (Vercel+Render); teste de "acordar" o free tier | URL pública roda a demo completa em H18 |
| A20 | Revisar testes de governança (README §52) e cobrir `no privilege union` com os 4 cards reais | `pytest -q` verde |

### Dev B

| ID | Tarefa | DoD |
|---|---|---|
| B12 | `OpenAICompatProvider` (json_schema/json_object + validação + 1 retry + timeout) e roteamento `small`/`strong` por env | com chave real, agentes geram textos; sem chave, cai no Mock |
| B13 | Fallback por chamada: exceção/timeout → `MockProvider` + evento `AGENT_FALLBACK_USED` | teste: provider que sempre falha → demo completa mesmo assim |
| B14 | Prompts + playbooks curtos por agente (`knowledge/playbooks/*.md`, ~300 tokens), todos com instrução de **não aprovar** e de referenciar apenas `source_ids` fornecidos | outputs LLM validam no schema em ≥ 9/10 execuções |
| B15 | Planner com LLM `small` para extração de intent/campos + fallback regex; fluxo `MISSING_INFO_REQUESTED` → `waiting_input` → `/input` → retoma | demanda sem valor → pergunta "Qual o valor solicitado?" e continua após resposta |
| B16 | Review: crítica LLM adicional (issues extras com severidade ≤ medium, nunca bloqueiam sozinhas); `grounding_ok`/`policy_ok` refletem warnings do executor | rodada 2 passa sempre no caso demo |
| B17 | Consolidator: `FinalResult` com resumo (LLM, 1 parágrafo), métricas, stress v1 vs v2, alternativas, pendências (inclui issues residuais e bloqueios), `sources`, `governance` (usuário, agentes, permissões usadas, decisões bloqueadas) | C renderiza sem transformar dados |
| B18 | `request_adjustment` com comentário do humano → constraint para o structuring (`human_comment`) → re-review → novo `HUMAN_REVIEW_REQUIRED` | testado ponta a ponta |
| B19 | Determinismo: `seed`/temperatura 0, `DEMO_MODE` força constraints do roteiro; `make demo` imprime timeline no terminal | 5 execuções seguidas com LLM real chegam ao mesmo estado final |
| B20 | Testes: `test_agents_schema.py` (schemas, evidence_ids, nenhum `approved`), `test_orchestrator_loop.py` (reabre o agente certo e os dependentes), `test_context_builder.py` | `pytest -q` verde |

### Dev C

| ID | Tarefa | DoD |
|---|---|---|
| C10 | `EvidencePanel`: lista `case.sources` (tipo, título, agente que acessou, hora), clique expande excerpt/campos; source chips clicáveis nos cards apontam para cá | jurado consegue "auditar" uma fonte em 1 clique |
| C11 | `GovernancePanel`: usuário, permissões, agentes selecionados com `allowed_data_domains`, lista de checagens (autorizado/bloqueado) a partir de `/audit`; destaque para o bloqueio do Structuring | frase "Structuring Agent tentou acessar dados financeiros — bloqueado" visível |
| C12 | `MetricsBar`: tempo total, agentes, tool calls, tokens por agente, fontes, inconsistências, loops, custo estimado | valores de `case.metrics` |
| C13 | Registry visual (P1): página/aba com os 4 Agent Cards (`/registry/agents`) | acessível pelo header |
| C14 | Fluxo `waiting_input`: quando `MISSING_INFO_REQUESTED`, exibir pergunta e campo de resposta → `POST /input` | testado com a demanda sem valor |
| C15 | Estados de erro/empty/loading discretos; `AGENT_FALLBACK_USED` vira chip "modo determinístico" (não esconder) | nada quebra visualmente se o LLM falhar |
| C16 | Polimento visual (README §54): paleta sóbria, cards, badges, animações rápidas, sem avatares, sem bolhas de chat; responsivo para 1366×768 (projetor) | revisão em conjunto no checkpoint 2 |
| C17 | `docs/demo-script.md` v1: narração de 3 min alinhada aos 9 passos do README §40 + o que clicar em cada tela | ensaiado 1× no checkpoint 2 |

---

## 5. Fase 3 — Polimento e entrega (H18 → H24)

Todos param de adicionar features em **H20**. Só bug fix, texto, ensaio, entregáveis.

| ID | Tarefa | Dono | DoD |
|---|---|---|---|
| F1 | Ensaio completo na URL pública, aba anônima, 3×; cronometrar | todos | < 4 min, sem falha |
| F2 | Congelar `main` (tag `v1.0-hackathon`); só hotfix com aprovação dos 3 | A | tag criada |
| F3 | Gravar vídeo da demo na URL pública (mesma versão da tag) | C (narração: quem apresenta) | link público |
| F4 | Slides: problema → limitação → ideia → diferencial → MVP → prova (prints da Tela 3 e do bloqueio de governança) → visão; nome da equipe e trilha em todas as páginas | C + B | link aberto sem login |
| F5 | Ficha/submissão + `docs/submission-checklist.md` preenchido (README §3) | A | checklist 100 % |
| F6 | `README.md`: seção "Como rodar" e "Como fazer deploy" atualizadas; `.env.example` conferido; grep por secrets | A | zero secrets no repo |
| F7 | Plano de contingência do dia: URL aquecida 5 min antes; `DEMO_MODE=true`; `LLM_PROVIDER=mock` como fallback em 1 variável; vídeo como último recurso | B | testado |

---

## 6. Pontos de integração (quem espera o quê)

```text
Fase 0  A/B → C : schemas + docs/api.md + event model     (C gera types.ts e events.demo.json)
Fase 1  A → B   : ToolGateway, calculations, registry     (B constrói agentes em cima)
        A → C   : stub A4 reproduzindo events.demo.json   (C integra HTTP real em H4)
        B → A   : orchestrator.run(case_id)                (A liga em POST /run)
        B → C   : orquestrador real                        (checkpoint 1)
Fase 2  A → B   : retriever/search_policy, telemetry hooks
        B → C   : FinalResult completo, waiting_input, fallback events
        A → C   : /audit, /registry, metrics
```

Se alguém ficar bloqueado > 30 min por dependência de outro, usa mock local e segue; o desbloqueio vira prioridade do outro no próximo checkpoint.

---

## 7. Definição de "pronto" do MVP (espelha README §53)

- [ ] Enviar demanda → squad aparece → 4 agentes com funções distintas executam
- [ ] Tool calls mostram `source_id` e checagem de permissão (incluindo 1 bloqueio visível)
- [ ] Review encontra `PRODUCTIVITY_ASSUMPTION` → Orquestrador reabre Risk (e Structuring dependente) → análise corrigida (stress muda)
- [ ] Resultado consolidado com fontes, riscos, pendências, governança, métricas
- [ ] Human Gate: solicitar ajuste funciona; aprovar encerra; caso **nunca** termina sem ele
- [ ] Todos os dados fictícios e sinalizados; disclaimers nas 3 posições (README §55)
- [ ] `pytest -q` verde; CI verde; URL pública sem login; 4 entregas na mesma versão

---

## 8. Backlog (só depois do checkpoint 2)

| Prioridade | Item | Dono sugerido |
|---|---|---|
| P1 | Registry visual (C13), tokens na UI (C12), RAG (A13) — já no plano | — |
| P2 | `USER-DEMO-002` com menos permissões (A17) | A |
| P2 | Múltiplos cases listáveis (`GET /cases`) com `sqlite3` | A |
| P2 | SSE `/cases/{id}/stream` no lugar do polling | A + C |
| P2 | Modo "Generalista vs Squad" (README §62) — só se houver medição real | B |
| P2 | Provider Anthropic | B |
| P2 | Dashboard de evals com golden case | B |
