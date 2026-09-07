# Agentic AI Architecture — Diagram Guide

This README explains the accompanying Mermaid diagram (`agentic_ai_polished.mermaid`): what each layer does, what tech typically implements it, and the design rules that keep the diagram readable.

How to view it: paste the `.mermaid` file contents into the [Mermaid Live Editor](https://mermaid.live), a Markdown renderer that supports Mermaid (GitHub, Notion, Obsidian), or a VS Code Mermaid preview extension.

---

## 1. Design principle: strictly top-to-bottom, no cycles

The diagram is drawn as a directed acyclic graph (DAG) on purpose. Every arrow points downward, from client request to final response, and again downward from an offline log to a retrained model. There are **no arrows that loop back up** into an earlier stage.

Why this matters: Mermaid's layout engine (`dagre`) ranks nodes vertically based on the longest path through the *entire* graph, including any loops. A single backward edge — like a "retry" arrow pointing back to an early node — turns local ranking into a cycle-breaking problem, and the engine can end up repositioning nodes that have nothing to do with the loop (this happened a few times while building this diagram: a retry edge or a "response delivered to client" edge accidentally created a cycle and pulled unrelated nodes out of place).

The two loops in this system (retry-on-failure, and retrain-on-feedback) are handled without cycles:
- **Retry** is drawn as one-way and terminal: it retries up to 2 times, then escalates. It does not loop back into the main path.
- **Retraining** lives entirely inside a separate `OFFLINE` subgraph, connected to the live path only by one-directional dotted "logged for analysis" edges. It improves the *next* deployed version, not the request currently in flight — so it's conceptually and visually a separate track, not a loop in the live path.

Where two nodes do need a two-way relationship (e.g., the Supervisor reading and writing to Memory), it's drawn as a single one-directional edge (`reads / writes`) rather than two opposing arrows, specifically to avoid creating a small cycle.

---

## 2. Agentic design patterns used in this architecture

"Agentic design patterns" is specific vocabulary (see Anthropic's *Building Effective Agents*, and the broader multi-agent literature). This diagram uses several of them — naming them explicitly here so the mapping from pattern to diagram is clear:

| Pattern | What it means | Where it appears |
|---|---|---|
| **Orchestrator–workers** | A central planner decomposes a task and delegates to specialist workers | Supervisor Agent (5) delegating to the four specialist agents (6) |
| **Routing** | Classify the input, then send it down the appropriate specialized path | The Model Router (LiteLLM/RouteLLM) picking a cost/quality tier; the semantic cache short-circuiting repeat queries |
| **Parallelization** | Run independent checks/subtasks concurrently, then aggregate | Input Guardrails (3) run their four checks in parallel; the Evaluation Layer (11) scores groundedness/safety/cost in parallel |
| **Evaluator–optimizer** | Generate, evaluate, and iteratively refine based on feedback | Output Guardrails fail → Retry loop, which injects the failure/critique back into context before re-attempting |
| **Reflection** | An agent (or a dedicated agent) critiques output before it's finalized | The Critic Agent within the Multi-Agent Team |
| **Tool use** | The model calls external functions/APIs rather than relying only on parametric knowledge | The entire Knowledge + Tools layer (7), mediated through the MCP Registry |
| **ReAct (Reason → Act → Observe)** | Each agent internally loops between reasoning about what to do next and acting on it | Happens *inside* each specialist agent box (Research, Data/Code, Action, Critic) — not drawn as a separate loop in the main diagram to avoid clutter. See the companion diagram `react_loop_detail.mermaid`, which draws this loop on its own — a real cycle is fine there since it's a small, self-contained diagram, not stitched into the main DAG. |
| **Prompt chaining** | Breaking a task into an explicit sequence of steps, each feeding the next | The overall top-to-bottom pipeline itself — gateway → guardrails → cache → agents → guardrails → response |

---

## 3. Layer-by-layer walkthrough

### 0 · Client Layer
The entry point, which varies by application type:
| Channel | Typical stack |
|---|---|
| Web App | React / Next.js |
| Mobile App | React Native, Swift (iOS), Kotlin (Android) |
| Direct API | REST, gRPC, or GraphQL for server-to-server integration |

All three converge on the same downstream pipeline — nothing after this layer needs to know which channel the request came from.

### 1 · User Query
The raw request, whatever form it arrives in (text, structured payload, etc.).

### 2 · Gateway
Handles authentication and traffic shaping before anything touches an agent.
- **API Gateway**: Kong, APISIX, AWS API Gateway
- **AuthN**: Okta, Auth0 (OIDC) — issues tenant + role claims used later for tool-level permission scoping
- Rate limiting / quota enforcement per tenant

### 3 · Input Guardrails
Runs in parallel, fails closed. If anything fails, the request is refused with a safe, logged response — it never fails silently.
- **PII / secrets detection**: Presidio
- **Safety classification**: Llama Guard
- **Prompt injection / jailbreak detection**: Lakera, Rebuff
- **Topical scoping** (is this even in-domain?): NeMo Guardrails

### 4 · Semantic Cache
Checks whether a semantically similar request has already been answered, to skip the (expensive) agent path entirely.
- Redis, GPTCache

### 5 · Supervisor Agent
The orchestrator. Decomposes the request into subtasks and delegates to the right specialist agent(s).
- LangGraph, CrewAI, OpenAI Agents SDK

### Budget & Loop Governor
Attached to the Supervisor. Agentic systems can run away — an agent stuck in a retry/tool-call loop, or a plan that balloons into hundreds of steps, silently burns cost and latency. This component enforces hard ceilings: max steps, max tokens, a wall-clock timeout, and a detector for repeated identical tool calls (the most common signature of a stuck agent).

### Memory Layer
Sits alongside the Supervisor, not in the main request path — it's infrastructure the Supervisor and other agents read from and write to throughout a run. See the dedicated **Memory Types** section below.

### 6 · Multi-Agent Team (A2A)
Specialist agents that split the work:
- **Research Agent** — retrieval and lookup
- **Data / Code Agent** — computation, data transforms
- **Action Agent** — takes real-world actions (tool calls, writes)
- **Critic Agent** — reviews the other agents' output before it moves on

They communicate using **A2A (Agent2Agent)**, an open protocol (originated at Google) that lets agents built on different frameworks exchange structured task/result messages — so a LangGraph agent and a CrewAI agent, for instance, can hand off work without a custom integration.

### Model Serving Layer
Every agent's reasoning ultimately calls an LLM. This layer is the shared backend:
- **Serving infra**: vLLM (self-hosted), AWS Bedrock, GCP Vertex AI, Azure OpenAI
- **Router**: LiteLLM or RouteLLM — picks a cost/quality tier per request (e.g., cheap model for simple steps, frontier model for hard reasoning)
- **Fallback / circuit breaker**: if the primary model provider fails or times out, requests circuit-break to a secondary model rather than failing the whole run

### 7 · Knowledge + Tools
What agents call to get information or take action:
- **Raw Document Store**: S3 / GCS / Azure Blob — where source documents (PDFs, wikis, etc.) live *before* ingestion. This feeds the RAG store via an offline batch pipeline (chunk → embed), not in the live request path.
- **RAG (vector retrieval)**: pgvector (Postgres extension), Qdrant, or Weaviate for the vector index, plus a cross-encoder reranker for precision
- **Knowledge Graph**: Neo4j or an RDF store, for entities and their relationships — used where grounded, explainable facts matter more than similarity search
- **MCP Registry**: declares every tool's schema, required scope, and auth in one place, so tool definitions aren't scattered across prompts
- **Tools**: web search, a code sandbox (E2B, gVisor — real isolation, not `exec`), and enterprise APIs/DB (commonly Postgres or MySQL for business data)

**Secrets Management**: tool credentials (API keys, DB passwords) are never hardcoded or embedded in prompts. The MCP Registry fetches short-lived, scoped credentials from a secrets manager (HashiCorp Vault, AWS Secrets Manager) per tool call.

**Agent (non-human) identity**: agents authenticate to tools using their *own* short-lived, scoped credentials (SPIFFE, or OAuth2 client-credentials) — not by reusing the end user's login. This keeps the blast radius of a compromised or misbehaving agent limited to exactly what that agent's role is scoped to, independent of what the requesting user could do directly.

**Idempotency**: every write action (payment, ticket creation, database write) carries an idempotency key, so a retried or duplicated tool call can't cause the same side effect twice — important once you have automatic retries in the loop.

### 8-9 · Risk Check + Human Approval
Any irreversible action (payment, sending a message, deleting data) routes through a human approval step before execution — typically a Slack or email interrupt card showing a diff/preview. Rejections go straight to the retry path.

### 10 · Output Guardrails
Validates the response before it's shown to anyone:
- **Groundedness / hallucination check**: RAGAS
- **Policy enforcement**: OPA (Open Policy Agent) — business rules, brand/legal constraints
- **Structured output validation**: Instructor, Outlines — for cases requiring strict JSON/schema conformance

### 11 · Evaluation Layer (online)
Scores every live response as it goes out — this is measurement, not a gate. It doesn't block the response; it logs metrics for later analysis.
- **Groundedness score**: RAGAS, Bespoke-MiniCheck
- **Safety/policy score**: Llama Guard, OPA
- **Latency/cost score**: Datadog, Prometheus

### 12-13 · Response + Delivery
The response returns through whichever channel the request came in on (Web, Mobile, or API) — drawn as separate delivery nodes rather than looping back into the Client Layer nodes, again to avoid a cycle.

---

## 4. Memory types

"Memory" isn't one thing — the diagram breaks it into five distinct kinds, each with a different job and different backing tech:

| Memory type | What it holds | Typical tech |
|---|---|---|
| **Working memory** | The current conversation and scratchpad for the active task | The model's context window itself, or Redis if it needs to spill outside the window |
| **Episodic memory** | A record of past interactions — what happened, when | Mem0, Zep |
| **Semantic memory** | Facts, entities, and general knowledge the system has accumulated | Shares infrastructure with the Knowledge + Tools layer — the vector DB (pgvector/Qdrant/Weaviate) and the Knowledge Graph (Neo4j) |
| **Procedural memory** | *How* to do things — learned tool-use routines, skills | Fine-tuned model weights, or cached/compiled prompt chains that encode a learned procedure |
| **State checkpoint** | Not memory of content, but of execution progress — lets a multi-step run resume after a crash | LangGraph checkpointer or Temporal, typically backed by Postgres |

---

## 5. The retry loop

If output guardrails fail, or a human rejects a proposed action, the system retries with the failure/critique injected back into context — capped at 2 attempts, after which it escalates rather than looping indefinitely.

---

## 6. Offline: observability → retraining → deployment

This is the slower loop that improves the *next* version of the system, not the current request.

**Signal collection**: every step is logged (OpenTelemetry tracing, aggregated in Langfuse or Arize Phoenix). Feedback signals — thumbs-down, human overrides, low evaluation scores — feed into a triage step.

**Triage before touching model weights** — cheapest fix first:
| Symptom | Fix | Tech |
|---|---|---|
| Bad instruction-following, formatting | Prompt/context optimization | DSPy, promptfoo |
| Wrong documents retrieved | Re-chunk, re-embed, tune retrieval | sentence-transformers |
| Wrong tool called or bad arguments | Rewrite tool descriptions / MCP schema | — |
| Consistent, systematic model errors | Fine-tune (LoRA/DPO) | Axolotl, Unsloth, TRL |

**Offline evaluation gate** (must pass before anything ships): golden-set accuracy, regression suite, automated red-teaming (PyRIT, Garak).

**CI/CD pipeline**: build → containerize (Docker) → provision infra (Terraform) — GitHub Actions, GitLab CI, or Jenkins.

**Registry**: MLflow or Weights & Biases as the model and prompt registry — Postgres for metadata, S3 for the actual artifacts (LoRA weights, checkpoints). Every version is immutable and traceable back to the training data and eval results that produced it.

**Progressive rollout** — a change never goes straight to 100% of traffic:
1. **Shadow mode**: the new version runs against real live traffic in parallel, but its output is discarded, not shown to users — pure risk-free measurement.
2. **Canary (~5%)**: a small slice of real traffic gets real responses from the new version.
3. **A/B test (~50%)**: a statistically significant split against the current version, gated on business KPIs, not just model metrics — typically managed with a feature-flag system like LaunchDarkly.
4. **Full rollout (100%)**: served through vLLM/Bedrock once the earlier stages are clean.

**Auto-rollback**: if a regression or SLO breach is detected at the canary, A/B, or full stage, it automatically rolls back and gets flagged for the next retraining cycle (this is intentionally a one-way terminal edge in the diagram, not a loop back into the retrain step, to keep the graph acyclic).

---

## 7. Where Postgres and S3 show up

Neither is drawn as one big central database — they're used in several specific places:

**Postgres**
- Backing the RAG vector store via the `pgvector` extension
- Backing state checkpoints (LangGraph checkpointer / Temporal)
- The enterprise APIs/DB tool an agent queries for business data
- MLflow's tracking/metadata backend

**S3** (or GCS / Azure Blob)
- Raw document storage, upstream of the RAG ingestion pipeline (chunk → embed)
- Model artifact storage for the registry (LoRA weights, fine-tuned checkpoints)
- Cold archival of logs/traces beyond what the observability platform keeps hot

Neither system stores raw credentials — those live in a dedicated secrets manager (Vault, AWS Secrets Manager), fetched short-lived and per-call rather than persisted anywhere.

---

## 8. Legend (colors used in the diagram)

| Color | Category |
|---|---|
| Slate/gray | Client layer |
| Blue | Core orchestration |
| Light blue | Agents / tools |
| Red | Guardrail or gate |
| Pink, bold border | Human-in-the-loop |
| Orange | Evaluation |
| Yellow | Observability |
| Purple | Retrain / deploy |
| Green | Knowledge stores (vector DB, knowledge graph, document store) |
