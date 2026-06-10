> **Status:** DUPLICATE — do not create. Already filed as [#1733](https://github.com/MervinPraison/PraisonAI/issues/1733) and [#1730](https://github.com/MervinPraison/PraisonAI/issues/1730) (closed; still reproduces on main @ ce976671 / 4.6.52). Consider reopening #1733.
>
> **Repo test baseline:** PraisonAI main @ ce976671 | Windows 10 | Python 3.13.2 | praisonai 4.6.52 | praisonaiagents 1.6.52

# Issue Draft 1 — Multi-Agent OpenAI Failure

**Repository:** https://github.com/MervinPraison/PraisonAI

## 1. Issue Title

**Short title:** Multi-agent `Agents`/`AgentTeam` fails with OpenAI sync streaming error

**Alternative titles:**
- `Agents.start()` fails: `Streaming is not supported in sync OpenAIAdapter`
- README multi-agent example broken with default OpenAI configuration
- `MultiAgentOutputConfig.stream=True` breaks sync OpenAI path

**Recommended final title:** `Multi-agent Agents/AgentTeam fails with OpenAI: sync streaming not supported (default stream=True)`

---

## 2. Classification

- Bug
- Regression
- Developer Experience

---

## 3. Severity

**Critical**

Single-agent onboarding works, but README step 2 (multi-agent) fails 100% of the time with OpenAI on sync paths. This blocks the natural progression from one agent to a team — a core onboarding flow.

---

## 4. Environment

```
OS:                 Windows 10 (AMD64)
Python:             3.13.2
PraisonAI:          4.6.52
PraisonAIAgents:    1.6.52
Installation:       editable from repo (main @ ce976671)
LLM provider:       OpenAI
Model:              gpt-4o-mini (default)
API key env:        OPENAI_API_KEY
```

---

## 5. Executive Summary

**What is broken:** `Agents(...).start()` and `AgentTeam(...).start()` fail on every task when using OpenAI with the default configuration. All tasks return `failed` with `None` results.

**Why it matters:** The README "Multi Agents" example is the second step in onboarding. New users who succeed with a single `Agent` hit a wall immediately after.

**Impact on new users:** After the first success, the next documented example fails silently (tasks marked failed, no useful output).

**Impact on OpenAI onboarding:** OpenAI is the default provider (`gpt-4o-mini`), but multi-agent — a flagship feature — does not work without undocumented workarounds.

---

## 6. Architecture Analysis (Broken Flow)

```
User (README step 2)
        |
        v
  Agents(agents=[a1, a2])
        |
        |  MultiAgentOutputConfig.stream defaults to True
        v
  Agents.start("prompt")
        |
        v
  _execute_with_agent_sync(..., stream=True)     <-- explicit stream=True
        |
        v
  Agent.chat(..., stream=True)                   <-- bypasses auto-fallback
        |
        |  chat_mixin fallback ONLY runs when stream is None
        |  (lines 622-652 in chat_mixin.py)
        v
  _execute_unified_chat_completion(stream=True)
        |
        v
  OpenAIAdapter.chat_completion(stream=True)     <-- sync path
        |
        v
  *** ValueError: Streaming is not supported ***
        in sync OpenAIAdapter
        |
        v
  task_status: {0: 'failed', 1: 'failed'}
```

**Failure point:** `unified_adapters.py:264-268` raises when `stream=True` on sync adapter. Multi-agent passes `stream=True` explicitly, so the single-agent fallback in `chat_mixin.py:623` never runs.

---

## 7. Expected Architecture

```
User
        |
        v
  Agents(agents=[a1, a2])
        |
        |  stream=None OR stream=False OR fallback on stream error
        v
  Agents.start("prompt")
        |
        v
  Agent.chat(..., stream=None)                   <-- auto-detect path
        |
        +--> try stream=True
        |         |
        |         +--> ValueError "Streaming not supported"
        |                   |
        |                   v
        +--> fallback stream=False --------------+
                                                  |
                                                  v
                                    OpenAIAdapter.chat_completion(stream=False)
                                                  |
                                                  v
                                    asyncio.run(achat_completion(...))  OK
                                                  |
                                                  v
                                    task completes, results returned
```

**Difference:** Single-agent uses `stream=None` and falls back gracefully. Multi-agent defaults to `stream=True` and never falls back.

---

## 8. Reproduction Steps

### Minimal Reproduction

```python
from praisonaiagents import Agent, Agents

a1 = Agent(instructions="Research briefly")
a2 = Agent(instructions="Summarize in one sentence")
result = Agents(agents=[a1, a2]).start("What is Python?")
print(result)
```

### Full Reproduction (README example)

```python
from praisonaiagents import Agent, Agents

research_agent = Agent(instructions="Research about AI")
summarise_agent = Agent(instructions="Summarise research agent's findings")
agents = Agents(agents=[research_agent, summarise_agent])
agents.start()
```

### CLI Reproduction

N/A — this is SDK-level. YAML multi-agent via `praisonai agents.yaml --framework praisonai` also fails for the same streaming reason once framework init is fixed.

---

## 9. Actual Result

```
[11:14:22] chat_mixin.py:968 ERROR Unified chat completion failed: Streaming is not supported in sync OpenAIAdapter. Use achat_completion() for streaming support.
┌────────────────────────────────── ⚠ Error ───────────────────────────────────┐
│ Unexpected error in chat: [llm] Streaming is not supported in sync           │
│ OpenAIAdapter. Use achat_completion() for streaming support. (agent: None,   │
│ run: 5f72e596-ddb2-405a-ae86-0cba264565d7)                                   │
└──────────────────────────────────────────────────────────────────────────────┘
[11:14:23] chat_mixin.py:968 ERROR Unified chat completion failed: Streaming is not supported in sync OpenAIAdapter. Use achat_completion() for streaming support.
┌────────────────────────────────── ⚠ Error ───────────────────────────────────┐
│ Unexpected error in chat: [llm] Streaming is not supported in sync           │
│ OpenAIAdapter. Use achat_completion() for streaming support. (agent: None,   │
│ run: f45a595a-9798-43d2-a258-95b5d64e6d53)                                   │
└──────────────────────────────────────────────────────────────────────────────┘
[11:14:23] chat_mixin.py:968 ERROR Unified chat completion failed: Streaming is not supported in sync OpenAIAdapter. Use achat_completion() for streaming support.
┌────────────────────────────────── ⚠ Error ───────────────────────────────────┐
│ Unexpected error in chat: [llm] Streaming is not supported in sync           │
│ OpenAIAdapter. Use achat_completion() for streaming support. (agent: None,   │
│ run: 86a9cae2-81d6-4b32-971a-436579682abb)                                   │
└──────────────────────────────────────────────────────────────────────────────┘
TEAM: {'task_status': {0: 'failed', 1: 'failed'}, 'task_results': {0: None, 1: None}}
```

Verified: `Agents(...).stream` defaults to `True`. Setting `OutputConfig(stream=False)` did not fix the failure in testing.

---

## 10. Expected Result

- Multi-agent execution completes sequentially
- Each agent calls OpenAI and returns text
- `task_status` shows success for all tasks
- README multi-agent example works with only `OPENAI_API_KEY` set

---

## 11. Root Cause Analysis

| Location | Role |
|----------|------|
| `src/praisonai-agents/praisonaiagents/config/feature_configs.py:958` | `MultiAgentOutputConfig.stream: bool = True` |
| `src/praisonai-agents/praisonaiagents/agents/agents.py:649-652` | Resolves `_stream = True` as default |
| `src/praisonai-agents/praisonaiagents/agents/agents.py:257-278` | `_execute_with_agent_sync(..., stream=stream)` passes explicit True |
| `src/praisonai-agents/praisonaiagents/agent/chat_mixin.py:623` | Fallback only when `stream is None` |
| `src/praisonai-agents/praisonaiagents/llm/unified_adapters.py:264-268` | Sync adapter rejects `stream=True` |

Single-agent `Agent.start()` works because it calls `chat()` with `stream=None`, triggering the try/fallback at lines 622-652.

---

## 12. Impact Analysis

| Audience | Impact |
|----------|--------|
| New users | High — README step 2 fails immediately after step 1 succeeds |
| OpenAI users | High — default provider, default model, default config |
| Multi-agent users | Critical — 100% failure on sync path |
| Windows users | Same as all platforms (not OS-specific) |
| YAML users | High — praisonai adapter multi-agent YAML also streams |
| Documentation readers | Misleading — docs imply multi-agent works out of the box |

---

## 13. Suggested Fix

### Quick Fix

Change default in `MultiAgentOutputConfig`:

```python
# feature_configs.py
stream: bool = False  # was True
```

### Proper Fix

Pass `stream=None` from multi-agent executor so single-agent fallback applies:

```python
# agents.py _execute_with_agent_sync
return executor_agent.chat(..., stream=None if stream else stream)
```

Or replicate the try/fallback block from `chat_mixin.py` in the multi-agent path.

### Long-Term Fix

- Implement sync streaming in `OpenAIAdapter.chat_completion()` via thread-safe async bridge, OR
- Make multi-agent execution async-native (`astart()`) and route through `achat_completion()`
- Align defaults: single-agent and multi-agent should share one streaming policy

---

## 14. Acceptance Criteria

- [ ] `Agents(agents=[a1, a2]).start("...")` succeeds with OpenAI + `OPENAI_API_KEY` only
- [ ] `AgentTeam(...).start()` succeeds with same setup
- [ ] README multi-agent example runs without modification
- [ ] No manual `stream=False` required for default OpenAI onboarding
- [ ] Existing unit tests pass
- [ ] New integration test added for multi-agent + OpenAI sync path

---

## 15. Regression Test Proposal

**Integration test:**
```python
@pytest.mark.live
def test_agents_start_openai_sync():
    a1 = Agent(instructions="Say hi")
    a2 = Agent(instructions="Say bye")
    result = Agents(agents=[a1, a2]).start("test")
    assert result["task_status"][0] != "failed"
```

**Unit test:** Assert `_execute_with_agent_sync` passes `stream=None` or that fallback triggers when sync adapter rejects streaming.

---

## 16. Additional Context

- **Related (not duplicate):** [#1788](https://github.com/MervinPraison/PraisonAI/issues/1788) — gateway stream relay; different code path
- **Related (not duplicate):** [#1392](https://github.com/MervinPraison/PraisonAI/issues/1392) — fragmented streaming architecture
- **Workaround:** None reliable found; single-agent only
- **Single-agent control test:** `Agent(...).start("Reply with exactly: PRAISON_OK")` → `PRAISON_OK` ✓

---
