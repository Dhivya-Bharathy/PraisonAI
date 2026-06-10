## Executive summary

| Framework | Install (fresh venv) | Single agent works? | Multi-agent works? | Steps to first success | vs PraisonAI |
|-----------|---------------------:|--------------------|--------------------|------------------------|--------------|
| **PraisonAI** (`praisonaiagents`) | ~171 s | **Yes** (~7 s run) | **No** (streaming sync error) | 2 (pip + code) | baseline |
| **OpenAI Agents SDK** | ~480 s | **Yes** (~14 s run) | **Yes** (~8 s, 2 agents) | 2 + async required | Faster multi-agent; slower install; async-only |
| **CrewAI** | ~1062 s (~18 min) | **Yes** (~3 s run) | **Yes** (~11 s) | 3+ (verbose Agent/Task/Crew) | Heavier install; multi-agent works |
| **LangChain** (`langchain-openai`) | ~81 s | **Yes** (~few s run) | N/A (manual chain) | 2 (not full agent framework) | Lightest install; not apples-to-apples agent API |

**Key finding:** PraisonAI matches or beats others on **single-agent** speed and **install size**, but is **alone in failing README multi-agent** on OpenAI sync path. CrewAI and OpenAI Agents SDK both complete multi-agent flows out of the box.

---

## Architecture overview (cross-framework)

### Package / layer model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ONBOARDING SURFACE LAYER                              │
├──────────────┬────────────────┬──────────────────┬───────────────────────────┤
│ PraisonAI    │ OpenAI Agents  │ CrewAI           │ LangChain                 │
│ README path  │ SDK            │                  │ (baseline)                │
├──────────────┼────────────────┼──────────────────┼───────────────────────────┤
│ pip install  │ pip install    │ pip install      │ pip install               │
│ praisonai    │ openai-agents  │ crewai           │ langchain-openai          │
│ agents       │                │                  │                           │
│ (+ praisonai │                │                  │                           │
│  for CLI)    │                │                  │                           │
└──────────────┴────────────────┴──────────────────┴───────────────────────────┘
         │                │                │                    │
         v                v                v                    v
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RUNTIME / ORCHESTRATION LAYER                       │
├──────────────┬────────────────┬──────────────────┬───────────────────────────┤
│ Agent        │ Agent +        │ Agent + Task +   │ ChatOpenAI.invoke()       │
│ Agents       │ Runner (async) │ Crew + Process   │ (single LLM call)         │
│ AgentTeam    │                │                  │                           │
└──────────────┴────────────────┴──────────────────┴───────────────────────────┘
         │                │                │                    │
         v                v                v                    v
┌─────────────────────────────────────────────────────────────────────────────┐
│                              LLM TRANSPORT LAYER                            │
├──────────────┬────────────────┬──────────────────┬───────────────────────────┤
│ OpenAIAdapter│ OpenAI HTTP    │ LiteLLM /        │ langchain-openai          │
│ (sync wraps  │ client (async  │ provider router  │ → OpenAI REST             │
│  async)      │ native)        │ (gpt-4o-mini)    │                           │
│ UnifiedLLM   │                │                  │                           │
│ Dispatcher   │                │                  │                           │
└──────────────┴────────────────┴──────────────────┴───────────────────────────┘
         │                │                │                    │
         v                v                v                    v
                    OpenAI API (gpt-4o-mini)
                    OPENAI_API_KEY
```

### Sync vs async entry points

| Framework | Primary onboarding API | Under the hood | Multi-agent default transport |
|-----------|------------------------|----------------|------------------------------|
| PraisonAI | `Agent.start()` sync | `chat()` → `OpenAIAdapter.chat_completion()` | Same sync path, **`stream=True` forced** |
| OpenAI Agents SDK | `Runner.run()` async | Native async OpenAI client | Async non-streaming or streaming OK |
| CrewAI | `crew.kickoff()` sync | Internal LLM calls (sync) | Sequential task graph, no sync-stream conflict |
| LangChain | `llm.invoke()` sync | HTTP client sync | N/A |

### Why PraisonAI multi-agent fails (architectural divergence)

Single-agent and multi-agent do **not** share the same LLM call policy:

```
SINGLE AGENT (works)
────────────────────
Agent.start(prompt)
    → chat(stream=None)                    # chat_mixin.py ~623
    → try stream=True
    → ValueError "Streaming not supported"
    → fallback stream=False                # chat_mixin.py ~643-646
    → OpenAIAdapter.chat_completion(stream=False)
    → asyncio.run(achat_completion(...))   # unified_adapters.py ~284
    → OK

MULTI-AGENT (fails)
───────────────────
Agents.start(prompt)
    → MultiAgentOutputConfig.stream = True # feature_configs.py ~958 (default)
    → _execute_with_agent_sync(..., stream=True)  # agents.py ~257-278
    → Agent.chat(stream=True)              # explicit — NO fallback branch
    → OpenAIAdapter.chat_completion(stream=True)
    → ValueError at unified_adapters.py ~264-268
    → task_status: failed
```

**Root architectural issue:** streaming fallback exists only when `stream is None` in `chat_mixin.py`. Multi-agent executor passes `stream=True` explicitly, bypassing the single-agent recovery path.

### PraisonAI two-package architecture (onboarding friction)

```
Developer README quickstart          Developer YAML/CLI path
         │                                    │
         v                                    v
  praisonaiagents (core SDK)            praisonai (wrapper)
  ├── Agent / Agents                  ├── CLI (main.py)
  ├── llm/openai_client.py            ├── AgentsGenerator
  ├── llm/unified_adapters.py         ├── framework_adapters/
  └── agents/agents.py                │   ├── praisonai_adapter
         │                            │   ├── crewai_adapter
         │                            │   └── autogen_adapter...
         │                            └── tool_resolver, deploy, UI...
         │                                    │
         └──────── same Agent class ──────────┘
                    (wrapper delegates to core)

Gap: README installs only praisonaiagents; YAML path requires praisonai + framework adapter init order (#1877).
```

---

## Detailed results

### 1. PraisonAI (`praisonaiagents`)

```powershell
pip install praisonaiagents
$env:OPENAI_API_KEY="sk-..."
```

**Single agent:**
```python
from praisonaiagents import Agent
agent = Agent(instructions="Reply with exactly the requested text")
agent.start("Reply with exactly: OK")  # → OK
```

| Metric | Fresh venv | System (editable install) |
|--------|------------|-------------------------|
| Install | ~171 s | n/a |
| Import | 0.26 s | 1.05 s |
| First run | 6.58 s | 29.8 s |

**Multi-agent (README example):**
```python
from praisonaiagents import Agent, Agents
agents = Agents(agents=[Agent(...), Agent(...)]).start("test")
# → {'task_status': {0: 'failed', 1: 'failed'}, ...}
# Error: Streaming is not supported in sync OpenAIAdapter
```

| Metric | Value |
|--------|------:|
| Multi-agent | **FAIL** (~3.6 s to fail) |

**Friction vs competitors:**
- Multi-agent broken (others work)
- Two packages if user needs CLI/YAML (`praisonai` vs `praisonaiagents`)
- README uses bash `export` only
- Async not required for single agent (pro vs OpenAI SDK)

#### Architecture (PraisonAI core)

**Key modules (onboarding path):**

| Module | Path | Role |
|--------|------|------|
| `Agent` | `praisonaiagents/agent/agent.py` | User-facing agent; default model `gpt-4o-mini` via `OPENAI_MODEL_NAME` |
| `Agents` | `praisonaiagents/agents/agents.py` | Multi-agent orchestrator; task queue + sequential execution |
| `chat_mixin` | `praisonaiagents/agent/chat_mixin.py` | LLM call loop; streaming fallback when `stream=None` |
| `OpenAIAdapter` | `praisonaiagents/llm/unified_adapters.py` | Sync wrapper over async; **rejects `stream=True`** on sync path |
| `MultiAgentOutputConfig` | `praisonaiagents/config/feature_configs.py` | Default `stream: bool = True` — triggers failure |
| `OpenAIClient` | `praisonaiagents/llm/openai_client.py` | Reads `OPENAI_API_KEY`, `OPENAI_API_BASE` |

**Single-agent execution graph:**

```
User code: Agent(instructions=...).start(prompt)
    │
    v
execution_mixin.start()
    │
    v
chat_mixin.chat(prompt, stream=None)
    │
    ├─[stream is None]─► try _execute_unified_chat_completion(stream=True)
    │                         │
    │                         ├─ fail: "Streaming not supported"
    │                         └─ fallback stream=False ─────────────┐
    │                                                                │
    └─[stream False]────────────────────────────────────────────────►│
                                                                     v
                                                    UnifiedLLMDispatcher
                                                                     │
                                                                     v
                                                    OpenAIAdapter.chat_completion(stream=False)
                                                                     │
                                                                     v
                                                    asyncio.run(achat_completion(...))
                                                                     │
                                                                     v
                                                    OpenAI HTTPS API  →  text response
```

**Multi-agent execution graph:**

```
User code: Agents(agents=[a1, a2]).start(prompt)
    │
    v
Agents.__init__  →  MultiAgentOutputConfig()  →  self.stream = True
    │
    v
For each Task in queue:
    │
    v
_execute_with_agent_sync(agent, task_prompt, task, tools, stream=True)
    │
    v
agent.chat(..., stream=True)          ← stream is NOT None → no fallback
    │
    v
_execute_unified_chat_completion(stream=True)
    │
    v
OpenAIAdapter.chat_completion(stream=True)
    │
    v
ValueError("Streaming is not supported in sync OpenAIAdapter")  →  task failed
```

**Config resolution chain:**

```
Environment                    Agent defaults
───────────                    ──────────────
OPENAI_API_KEY        ──►      OpenAIClient / OpenAIAdapter auth
OPENAI_MODEL_NAME     ──►      Agent._get_default_model() → gpt-4o-mini
MODEL_NAME            ──►      resolve_llm_endpoint() (wrapper CLI)
OPENAI_BASE_URL       ──►      optional custom endpoint

Not loaded automatically in SDK:  .env file (CLI wrapper loads via load_dotenv)
```

**Wrapper layer (when using CLI/YAML):**

```
praisonai agents.yaml
    │
    v
cli/main.py  →  AgentsGenerator(agent_file, framework="", ...)
    │
    v
__init__: framework_adapter = create("")     ← BUG: empty before YAML read (#1877)
    │
    v (expected)
generate_crew_and_kickoff()
    → load YAML → framework = config.get("framework", "crewai")
    → praisonai_adapter.run(config, ...)
    → Agent instances → same OpenAI path as SDK
```

---

### 2. OpenAI Agents SDK (`openai-agents`)

```powershell
pip install openai-agents
$env:OPENAI_API_KEY="sk-..."
```

**Single agent:**
```python
import asyncio
from agents import Agent, Runner

async def main():
    agent = Agent(name="Assistant", instructions="...", model="gpt-4o-mini")
    result = await Runner.run(agent, "Reply with exactly: OK")
asyncio.run(main())  # → OK
```

| Metric | Value |
|--------|------:|
| Install | ~480 s |
| Import | 14.64 s |
| First run | 13.58 s |

**Multi-agent (sequential, 2 agents):**
| Metric | Value |
|--------|------:|
| Multi-agent | **PASS** (~8.1 s) |

**Friction vs PraisonAI:**
- Must use `asyncio` / `async def` (harder for beginners)
- Slower import and install
- **Multi-agent works** without extra flags
- Official OpenAI package — clear key setup story

#### Architecture (OpenAI Agents SDK)

**Key components:**

| Component | Role |
|-----------|------|
| `Agent` | Instructions + model binding (`model="gpt-4o-mini"`) |
| `Runner` | Async execution engine; tool loop + handoffs |
| `asyncio.run()` | Required entry point for scripts |

**Single-agent execution graph:**

```
User code: asyncio.run(Runner.run(agent, prompt))
    │
    v
Runner.run()  [async]
    │
    v
OpenAI async client  →  chat.completions (non-stream or stream, native async)
    │
    v
RunResult.final_output
```

**Multi-agent (benchmark pattern — sequential Runner calls):**

```
Runner.run(agent_a, prompt)  ──async──► OpenAI API  →  output_a
Runner.run(agent_b, prompt)  ──async──► OpenAI API  →  output_b
```

No sync adapter layer — async is canonical, so no `stream=True` on sync wrapper conflict.

**vs PraisonAI:** OpenAI SDK never wraps async behind a sync adapter that rejects streaming. PraisonAI's `OpenAIAdapter.chat_completion` is sync-first with explicit streaming prohibition.

---

### 3. CrewAI

```powershell
pip install crewai
$env:OPENAI_API_KEY="sk-..."
```

**Single agent:**
```python
from crewai import Agent, Task, Crew
agent = Agent(role='...', goal='...', backstory='...', llm='gpt-4o-mini', verbose=False)
task = Task(description='Reply with exactly: OK', expected_output='OK', agent=agent)
result = Crew(agents=[agent], tasks=[task], verbose=False).kickoff()  # → OK
```

| Metric | Value |
|--------|------:|
| Install | ~1062 s (**longest**) |
| Import | 16.44 s |
| First run | 2.69 s |

**Multi-agent (2 agents, sequential process):**
| Metric | Value |
|--------|------:|
| Multi-agent | **PASS** (~11.1 s) |

**Friction vs PraisonAI:**
- **Much heavier install** (~18 min vs ~3 min)
- More boilerplate (`role`, `goal`, `backstory`, `Task`, `Crew`)
- Tracing popup on first run (extra noise)
- **Multi-agent works** on OpenAI default path

#### Architecture (CrewAI)

**Key components:**

| Component | Role |
|-----------|------|
| `Agent` | Role/goal/backstory → system prompt assembly |
| `Task` | Unit of work + `expected_output` |
| `Crew` | Binds agents + tasks + `Process` (sequential/hierarchical) |
| LLM backend | Typically LiteLLM routing to OpenAI for `llm='gpt-4o-mini'` |

**Single-agent execution graph:**

```
Crew(agents=[agent], tasks=[task]).kickoff()
    │
    v
Task execution loop (sync)
    │
    v
Agent LLM call  →  LiteLLM / OpenAI  →  task output
    │
    v
CrewOutput.raw
```

**Multi-agent execution graph (sequential process):**

```
Crew(agents=[a1, a2], tasks=[t1, t2], process=Process.sequential).kickoff()
    │
    v
Task 1 (agent a1)  ──► LLM call  ──► context for Task 2
    │
    v
Task 2 (agent a2)  ──► LLM call  ──► final output
```

**vs PraisonAI:** CrewAI uses sync LLM calls end-to-end for `kickoff()`; no separate "output config stream default" that forces sync-incompatible streaming on the OpenAI path.

---

### 4. LangChain OpenAI (LLM invoke baseline)

```powershell
pip install langchain-openai
```

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
ChatOpenAI(model="gpt-4o-mini").invoke([HumanMessage(content="Reply with exactly: OK")])
```

| Metric | Value |
|--------|------:|
| Install | ~81 s |
| First run | ~few s |

**Note:** Not a full agent framework — included as lightweight OpenAI baseline. No built-in multi-agent.

#### Architecture (LangChain baseline)

**Stack depth:** shallowest — one LLM class, one HTTP round-trip.

```
ChatOpenAI(model="gpt-4o-mini").invoke([HumanMessage(...)])
    │
    v
langchain-openai  →  OpenAI REST  →  AIMessage
```

No agent loop, tool registry, or multi-agent orchestrator. Included to show minimum install surface (~81 s) vs full agent frameworks.

---

## Comparison matrix

| Criterion | PraisonAI | OpenAI Agents SDK | CrewAI | LangChain |
|-----------|-----------|-------------------|--------|-----------|
| **pip install time** | Good (~3 min) | Slow (~8 min) | Worst (~18 min) | Best (~1.4 min) |
| **Lines for first agent** | 3 | 8+ (async) | 8+ | 4 |
| **Multi-agent out of box** | **Fail** | Pass | Pass | N/A |
| **Windows env docs** | Weak | OK | OK | OK |
| **Sync API for beginners** | Yes | No (async) | Yes | Yes |
| **README accuracy** | Multi-agent wrong | Accurate | Accurate | Accurate |

### Architecture comparison

| Dimension | PraisonAI | OpenAI Agents SDK | CrewAI | LangChain |
|-----------|-----------|-------------------|--------|-----------|
| **Layers to first LLM call** | 4 (Agent → chat_mixin → Dispatcher → Adapter) | 2 (Runner → async client) | 3 (Crew → Task → LLM) | 1 (ChatOpenAI) |
| **Default transport** | Sync adapter wrapping async | Native async | Sync (LiteLLM) | Sync HTTP |
| **Streaming default (multi-agent)** | `True` (breaks sync OpenAI) | N/A (async-safe) | Framework-internal | N/A |
| **Multi-agent model** | `Agents` task queue, shared `stream` flag | Handoffs / sequential `Runner` | `Crew` + `Process` graph | Manual chaining |
| **Package split** | `praisonaiagents` + optional `praisonai` | Single `openai-agents` | Single `crewai` | Modular extras |
| **`.env` auto-load (SDK path)** | No (env vars only) | No (env vars only) | Yes (common pattern) | Yes |
| **Framework adapter indirection** | Yes (`praisonai` wrapper) | No | No | No |

### Architectural fix options (multi-agent #1733)

| Option | Change | Pros | Cons |
|--------|--------|------|------|
| **A. Default off** | `MultiAgentOutputConfig.stream = False` | One-line; matches sync onboarding | Loses streaming UX for async users |
| **B. Pass `None`** | Multi-agent executor passes `stream=None` into `chat()` | Reuses existing fallback in `chat_mixin.py` | Implicit behavior; harder to reason about |
| **C. Capability probe** | Before stream, check adapter supports sync stream | Correct for all adapters | More code in hot path |
| **D. Async multi-agent** | `Agents.astart()` as recommended multi path | Aligns with OpenAI SDK | Breaking doc change; async required |

**Recommended:** **A + B** — default `stream=False` for sync `Agents.start()`, pass `stream=None` when unset so single-agent and multi-agent share fallback semantics.

---

## Where PraisonAI is harder than others (file issues)

Already filed / tracked:

| Gap | vs competitor | Issue |
|-----|---------------|-------|
| Multi-agent fails on OpenAI | OpenAI SDK & CrewAI work | [#1733](https://github.com/MervinPraison/PraisonAI/issues/1733) (reopen) |
| YAML `praisonai agents.yaml` | CrewAI YAML path exists | [#1877](https://github.com/MervinPraison/PraisonAI/issues/1877) |

**New issues to consider from Task A:**

### Enhancement: Match competitor install story — document `praisonaiagents` vs full `praisonai`

- OpenAI SDK: one package `openai-agents`
- CrewAI: one package `crewai`
- PraisonAI: README splits `praisonaiagents` vs `praisonai` without decision tree

### Enhancement: Reduce import cold-start vs OpenAI Agents SDK

- PraisonAI fresh import: 0.26–1 s (good)
- OpenAI Agents SDK import: 14.6 s (PraisonAI wins on import)

### Enhancement: Simplify multi-agent to match CrewAI reliability

- CrewAI: `Crew(agents=[...], tasks=[...]).kickoff()` works
- PraisonAI: `Agents(...).start()` fails — **#1 onboarding regression vs peers**

---

## Simplification recommendations

### Product / docs

1. **Fix multi-agent streaming default** — brings PraisonAI to parity with CrewAI/OpenAI SDK on README step 2.
2. **Single-package quickstart** — document: “Start with `pip install praisonaiagents`; add `praisonai` only for CLI/YAML.”
3. **Windows + `.env` in README** — match what LangChain/CrewAI docs assume.
4. **Do not optimize install vs CrewAI** — PraisonAI already wins (~3 min vs ~18 min).
5. **Optional async quickstart** — add `agent.start()` async example for users comparing to OpenAI Agents SDK.

### Architecture alignment

6. **Unify LLM call policy** — single-agent fallback (`stream=None` → retry non-stream) should apply to multi-agent executor paths in `agents.py`.
7. **Lazy framework adapter init** — defer `AgentsGenerator` adapter creation until after YAML `framework` is read (#1877).
8. **Separate output config from transport** — `MultiAgentOutputConfig.stream` should mean “display streaming to user”, not “force sync-incompatible API flag”.
9. **SDK `.env` helper** — optional `load_dotenv()` in `praisonaiagents` import or documented one-liner for parity with CrewAI onboarding.

---

## Raw benchmark commands (repeatable)

See `_bench_*` venvs under repo root (local only, not for commit):

- `_bench_praisonai/`
- `_bench_crewai/`
- `_bench_openai_agents/`
- `_bench_langchain/`
