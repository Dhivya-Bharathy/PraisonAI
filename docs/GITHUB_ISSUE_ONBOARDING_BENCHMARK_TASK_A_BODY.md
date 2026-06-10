## Summary

Competitive **OpenAI-only onboarding benchmark** (Task A) comparing PraisonAI (`praisonaiagents`) against OpenAI Agents SDK, CrewAI, and LangChain on the same Windows machine with fresh venvs and `gpt-4o-mini`.

**Key finding:** PraisonAI matches or beats peers on **single-agent** install speed and simplicity, but is the **only framework where README multi-agent fails** on the default OpenAI sync path. CrewAI and OpenAI Agents SDK both complete multi-agent flows out of the box.

**Full report:** [docs/ONBOARDING_BENCHMARK_TASK_A.md](https://github.com/Dhivya-Bharathy/PraisonAI/blob/docs/openai-onboarding-issue-drafts/docs/ONBOARDING_BENCHMARK_TASK_A.md)

---

## Environment

| Item | Value |
|------|-------|
| OS | Windows 10 (AMD64) |
| Python | 3.13.2 |
| PraisonAI | 4.6.52 / praisonaiagents 1.6.52 |
| LLM | OpenAI `gpt-4o-mini` via `OPENAI_API_KEY` |
| Repo baseline | `main` @ `ce976671` |
| Install method | Fresh venv per framework (`pip install`) |

---

## Executive summary

| Framework | Install (fresh venv) | Single agent | Multi-agent | Steps to first success |
|-----------|---------------------:|--------------|-------------|------------------------|
| **PraisonAI** | ~171 s (~3 min) | **Yes** (~7 s) | **No** (streaming sync error) | 2 (pip + code) |
| **OpenAI Agents SDK** | ~480 s (~8 min) | **Yes** (~14 s) | **Yes** (~8 s, 2 agents) | 2 + async required |
| **CrewAI** | ~1062 s (~18 min) | **Yes** (~3 s) | **Yes** (~11 s) | 3+ (Agent/Task/Crew) |
| **LangChain** | ~81 s | **Yes** | N/A | 2 (LLM baseline, not full agent API) |

---

## Cross-framework architecture (onboarding stack)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ONBOARDING SURFACE LAYER                              │
├──────────────┬────────────────┬──────────────────┬───────────────────────────┤
│ PraisonAI    │ OpenAI Agents  │ CrewAI           │ LangChain                 │
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

---

## Error 1 — Multi-agent fails (CRITICAL — #1 onboarding regression)

### Reproduction

```python
from praisonaiagents import Agent, Agents

research_agent = Agent(instructions="Research about AI")
summarise_agent = Agent(instructions="Summarise research agent's findings")
agents = Agents(agents=[research_agent, summarise_agent])
result = agents.start("What is Python?")
print(result)
```

### Actual error output

```
[11:14:22] chat_mixin.py:968 ERROR Unified chat completion failed: Streaming is not supported in sync OpenAIAdapter. Use achat_completion() for streaming support.
┌────────────────────────────────── ⚠ Error ───────────────────────────────────┐
│ Unexpected error in chat: [llm] Streaming is not supported in sync           │
│ OpenAIAdapter. Use achat_completion() for streaming support. (agent: None,   │
│ run: 5f72e596-ddb2-405a-ae86-0cba264565d7)                                   │
└──────────────────────────────────────────────────────────────────────────────┘
(repeated for each agent/task)
TEAM: {'task_status': {0: 'failed', 1: 'failed'}, 'task_results': {0: None, 1: None}}
```

### Control test (single agent — PASSES)

```python
from praisonaiagents import Agent
Agent(instructions="Reply with exactly the requested text").start("Reply with exactly: OK")
# → OK
```

### Architectural divergence (why single works, multi fails)

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
    → MultiAgentOutputConfig.stream = True # feature_configs.py:958 (default)
    → _execute_with_agent_sync(..., stream=True)  # agents.py:257-278, 1290
    → Agent.chat(stream=True)              # explicit — NO fallback branch
    → OpenAIAdapter.chat_completion(stream=True)
    → ValueError at unified_adapters.py:264-268
    → task_status: failed
```

**Root cause:** streaming fallback in `chat_mixin.py` only runs when `stream is None`. Multi-agent executor passes `stream=True` explicitly.

### PraisonAI single-agent execution graph

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

### PraisonAI multi-agent execution graph

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

### Code locations

| Location | Role |
|----------|------|
| `src/praisonai-agents/praisonaiagents/config/feature_configs.py:958` | `MultiAgentOutputConfig.stream: bool = True` |
| `src/praisonai-agents/praisonaiagents/agents/agents.py:649-652, 790, 1290` | Resolves and passes `_stream = True` |
| `src/praisonai-agents/praisonaiagents/agents/agents.py:257-278` | `_execute_with_agent_sync(..., stream=stream)` |
| `src/praisonai-agents/praisonaiagents/agent/chat_mixin.py:623` | Fallback only when `stream is None` |
| `src/praisonai-agents/praisonaiagents/llm/unified_adapters.py:264-268` | Sync adapter rejects `stream=True` |

**Tracked:** [#1733](https://github.com/MervinPraison/PraisonAI/issues/1733) (closed; still reproduces on 4.6.52)

**Recommended fix (A + B):** default `stream=False` for sync `Agents.start()`, pass `stream=None` when unset so multi-agent reuses single-agent fallback.

---

## Error 2 — YAML CLI fails before execution

### Reproduction

```powershell
pip install praisonai
$env:OPENAI_API_KEY="sk-..."
praisonai agents.yaml
```

### Actual error

```
ValueError: Unknown praisonai.framework_adapters plugin: ''. Available: ['ag2', 'autogen', 'autogen_v4', 'crewai', 'praisonai']
```

### Architecture (init order bug)

```
praisonai agents.yaml
    │
    v
cli/main.py  →  AgentsGenerator(agent_file, framework="", ...)
    │
    v
__init__: framework_adapter = create("")     ← BUG: empty before YAML read
    │
    v (expected)
generate_crew_and_kickoff()
    → load YAML → framework = config.get("framework", "crewai")
    → praisonai_adapter.run(config, ...)
```

**Tracked:** [#1877](https://github.com/MervinPraison/PraisonAI/issues/1877)

---

## Error 3 — Doctor crashes on Windows default console

### Reproduction

```powershell
praisonai doctor env
```

### Actual error

```
ERROR: Doctor error: 'charmap' codec can't encode characters in position 25-94: character maps to <undefined>
UnicodeEncodeError: 'charmap' codec can't encode characters ...
```

**Workaround:** `praisonai doctor env --json` works.

**Tracked:** [#1878](https://github.com/MervinPraison/PraisonAI/issues/1878)

---

## Error 4 — CLI masks YAML errors (UnboundLocalError)

When YAML execution fails, secondary crash:

```
UnboundLocalError: cannot access local variable 'print' where it is not associated with a value
```

(`from rich import print` inside `cli/main.py` shadows module-level `print`)

**Tracked:** [#1879](https://github.com/MervinPraison/PraisonAI/issues/1879)

---

## Error 5 — Broken workflow example

```python
# examples/python/workflows/simple_workflow.py
from praisonaiagents import Workflow  # imports Workflow
workflow = AgentFlow(...)              # uses AgentFlow → NameError
```

```
NameError: name 'AgentFlow' is not defined
```

**Tracked:** [#1880](https://github.com/MervinPraison/PraisonAI/issues/1880)

---

## PraisonAI two-package architecture (onboarding friction)

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
```

Competitors use one package (`openai-agents`, `crewai`). PraisonAI README does not document when to add `praisonai`.

**Docs tracked:** [PraisonAIDocs #504](https://github.com/MervinPraison/PraisonAIDocs/issues/504)

---

## Comparison matrix

| Criterion | PraisonAI | OpenAI Agents SDK | CrewAI | LangChain |
|-----------|-----------|-------------------|--------|-----------|
| pip install time | Good (~3 min) | Slow (~8 min) | Worst (~18 min) | Best (~1.4 min) |
| Lines for first agent | 3 | 8+ (async) | 8+ | 4 |
| Multi-agent out of box | **Fail** | Pass | Pass | N/A |
| Windows env docs | Weak | OK | OK | OK |
| Sync API for beginners | Yes | No (async) | Yes | Yes |
| README accuracy | Multi-agent wrong | Accurate | Accurate | Accurate |

### Architecture comparison

| Dimension | PraisonAI | OpenAI Agents SDK | CrewAI | LangChain |
|-----------|-----------|-------------------|--------|-----------|
| Layers to first LLM call | 4 | 2 | 3 | 1 |
| Default transport | Sync adapter wrapping async | Native async | Sync (LiteLLM) | Sync HTTP |
| Streaming default (multi-agent) | `True` (breaks sync OpenAI) | Async-safe | Framework-internal | N/A |
| Multi-agent model | `Agents` task queue | Handoffs / sequential `Runner` | `Crew` + `Process` | Manual chaining |
| Package split | `praisonaiagents` + optional `praisonai` | Single package | Single package | Modular extras |
| `.env` auto-load (SDK path) | No | No | Yes (common) | Yes |

---

## Simplification recommendations

### Product / docs
1. Fix multi-agent streaming default — parity with CrewAI/OpenAI SDK on README step 2
2. Single-package quickstart: `pip install praisonaiagents`; add `praisonai` only for CLI/YAML
3. Windows + `.env` in README
4. Optional async quickstart for OpenAI Agents SDK comparers

### Architecture alignment
5. Unify LLM call policy across single- and multi-agent paths in `agents.py`
6. Lazy framework adapter init (#1877)
7. Decouple `MultiAgentOutputConfig.stream` (display UX) from transport-level streaming flag
8. SDK `.env` helper or documented one-liner

---

## Acceptance criteria (onboarding parity)

- [ ] `Agents(agents=[a1, a2]).start("...")` succeeds with OpenAI + `OPENAI_API_KEY` only
- [ ] README multi-agent example works unmodified on Windows
- [ ] Competitive benchmark re-run shows multi-agent **Pass**
- [ ] Package split documented with decision tree
- [ ] Windows env var setup documented alongside bash `export`

---

## Related issues

| Issue | Topic |
|-------|-------|
| [#1733](https://github.com/MervinPraison/PraisonAI/issues/1733) | Multi-agent streaming (reopen) |
| [#1730](https://github.com/MervinPraison/PraisonAI/issues/1730) | Same class of failure |
| [#1877](https://github.com/MervinPraison/PraisonAI/issues/1877) | YAML framework init |
| [#1878](https://github.com/MervinPraison/PraisonAI/issues/1878) | Doctor Windows Unicode |
| [#1879](https://github.com/MervinPraison/PraisonAI/issues/1879) | CLI print shadow |
| [#1880](https://github.com/MervinPraison/PraisonAI/issues/1880) | Broken workflow example |
| [PraisonAIDocs #504](https://github.com/MervinPraison/PraisonAIDocs/issues/504) | Docs gaps |
| [#1392](https://github.com/MervinPraison/PraisonAI/issues/1392) | Fragmented streaming architecture |
