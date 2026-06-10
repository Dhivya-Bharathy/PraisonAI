> **Status:** Draft for manual review. Do NOT auto-create.
>
> **Repo test baseline:** PraisonAI main @ ce976671 | Windows 10 | Python 3.13.2 | praisonai 4.6.52 | praisonaiagents 1.6.52

# Issue Draft 6 — OpenAI Onboarding Documentation Gaps

**Repository:** https://github.com/MervinPraison/PraisonAIDocs

> Also affects [PraisonAI README](https://github.com/MervinPraison/PraisonAI/blob/main/README.md) — recommend cross-repo sync after docs fix.

## 1. Issue Title

**Short title:** OpenAI quickstart docs missing Windows setup and broken multi-agent/YAML paths

**Alternative titles:**
- 5-minute OpenAI onboarding fails due to documentation gaps
- README uses bash-only env vars; multi-agent step 2 undocumented failure

**Recommended final title:** `Docs: OpenAI onboarding guide — Windows env vars, package split, and broken multi-agent/YAML paths`

---

## 2. Classification

- Documentation
- Developer Experience

---

## 3. Severity

**High**

Documentation actively guides users into failing paths (multi-agent, YAML) without Windows instructions or troubleshooting.

---

## 4. Environment

Documentation evaluated against:
- PraisonAI README @ `ce976671`
- `examples/doctor/README.md`
- `examples/README.md`
- docs.praison.ai (referenced links)

Test runtime: Windows 10, Python 3.13.2, OpenAI gpt-4o-mini

---

## 5. Executive Summary

**What is broken:** Documentation presents a sub-5-minute OpenAI onboarding that only works for single-agent Python API. Multi-agent, YAML, and Windows setup steps are incomplete or incorrect.

**Why it matters:** New developers on Windows (large user base) cannot follow README verbatim.

**Impact on OpenAI onboarding:** Step 1 works; steps 2+ fail or require undocumented flags.

---

## 6. Architecture Analysis (Documentation vs Reality)

```
README promises:
  pip install praisonaiagents          ✓ works
  export OPENAI_API_KEY=...            ✗ bash-only (Windows fails)
  Agent(...).start(...)                ✓ works
  Agents(...).start()                  ✗ fails (Issue #1)
  praisonai agents.yaml                ✗ fails (Issue #2)

Missing doc layers:
  - Windows PowerShell env var syntax
  - praisonaiagents vs praisonai package split
  - .env file loading (CLI yes, SDK no)
  - doctor env Windows workaround
  - --framework praisonai YAML flag
```

---

## 7. Expected Architecture (Documentation Flow)

```
Install (pick one path clearly documented)
  ├── pip install praisonaiagents  → Python API
  └── pip install praisonai        → CLI + YAML

Configure (all platforms)
  ├── .env file (recommended)
  ├── bash: export OPENAI_API_KEY=
  ├── PowerShell: $env:OPENAI_API_KEY=
  └── cmd: set OPENAI_API_KEY=

Verify
  └── praisonai doctor env --json   (Windows-safe)

Run (progressive)
  1. Single Agent ✓
  2. Multi Agent (after Issue #1 fix)
  3. YAML (after Issue #2 fix)
```

---

## 8. Reproduction Steps

Follow README verbatim on Windows PowerShell:

```powershell
pip install praisonaiagents
export OPENAI_API_KEY="your-api-key"   # FAILS: export not recognized
```

Follow README multi-agent step:

```python
from praisonaiagents import Agent, Agents
# ... fails with streaming error (Issue #1)
```

Follow README YAML step:

```bash
praisonai agents.yaml   # FAILS without --framework (Issue #2)
```

Follow examples/doctor/README.md:

```bash
praisonai doctor --only python_version,openai_api_key
# FAILS: --only flag removed; use subcommands
```

---

## 9. Actual Result

- Windows users cannot set env var with `export`
- Multi-agent README example returns all tasks failed
- YAML README example crashes on framework init
- Doctor example docs reference removed CLI flags

---

## 10. Expected Result

Copy-paste README instructions work on Windows, macOS, and Linux for the full OpenAI onboarding path.

---

## 11. Root Cause Analysis (Documentation Sources)

| Source | Problem |
|--------|---------|
| README.md:94-95 | `export OPENAI_API_KEY` — bash only |
| README.md:248-256 | Multi-agent example — fails at runtime |
| README.md:420-422 | `praisonai agents.yaml` — missing `--framework` note |
| README.md:90-105 | Only documents `praisonaiagents`, not when to install `praisonai` |
| examples/doctor/README.md:38,58 | Documents `--only` flag removed from CLI |
| SDK | `praisonaiagents` does not auto-load `.env`; not documented |

---

## 12. Impact Analysis

| Audience | Impact |
|----------|--------|
| New users | High — first-hour frustration |
| Windows users | Critical — env var step fails immediately |
| OpenAI users | High — primary documented provider |
| Documentation readers | Misleading success claims ("Under 1 Minute") |

---

## 13. Suggested Fix

See **Proposed Documentation** sections below.

---

## 14. Acceptance Criteria

- [ ] Windows PowerShell + cmd env var instructions added
- [ ] Package choice guide: `praisonaiagents` vs `praisonai`
- [ ] `.env` setup documented for both SDK and CLI
- [ ] Multi-agent example marked or fixed after Issue #1
- [ ] YAML example updated with `--framework praisonai` or fixed after Issue #2
- [ ] Doctor docs updated to subcommand model
- [ ] Troubleshooting section for common OpenAI errors

---

## 15. Regression Test Proposal

Docs CI: link checker + "copy-paste command" validation on Windows runner.

---

## 16. Additional Context

- **Related:** [PraisonAIDocs #427](https://github.com/MervinPraison/PraisonAIDocs/issues/427) — broader Windows deployment (closed)
- **Related:** [PraisonAIDocs #446](https://github.com/MervinPraison/PraisonAIDocs/issues/446) — Gateway charmap (closed); pattern applies to doctor
- **Blocked by code fixes:** Issues #1 and #2 should land before docs claim those paths work

---

## Current Documentation

From PraisonAI README (`README.md` lines 90-105):

```bash
pip install praisonaiagents
export OPENAI_API_KEY="your-api-key"
```

```python
from praisonaiagents import Agent

agent = Agent(instructions="You are a senior data analyst.")
agent.start("Analyze the top 3 tech trends of 2026 and format as a markdown table.")
```

From README multi-agent (lines 248-256):

```python
from praisonaiagents import Agent, Agents

research_agent = Agent(instructions="Research about AI")
summarise_agent = Agent(instructions="Summarise research agent's findings")
agents = Agents(agents=[research_agent, summarise_agent])
agents.start()
```

From README YAML (lines 420-422):

```bash
praisonai agents.yaml
```

From `examples/doctor/README.md` (lines 57-58):

```bash
praisonai doctor --only python_version,openai_api_key
```

---

## Problem

1. **`export` is bash-only** — fails on Windows PowerShell/cmd without translation.
2. **Two packages, one story** — quickstart installs `praisonaiagents` but YAML/CLI requires `praisonai`; not explained.
3. **No `.env` guidance** — SDK does not call `load_dotenv()`; users must export manually (undocumented).
4. **Multi-agent example broken** — documented as working; fails with OpenAI on latest main.
5. **YAML example broken** — `praisonai agents.yaml` fails without `--framework praisonai`.
6. **Doctor docs stale** — `--only` flag removed; CLI now uses `praisonai doctor env`.

---

## Proposed Documentation

New page: `docs/getting-started/openai-quickstart.mdx` (PraisonAIDocs)

Sections:
1. Install (choose your path)
2. Set API key (all platforms)
3. Verify setup (`praisonai doctor env --json`)
4. First agent (Python)
5. First CLI agent
6. Multi-agent (after fix)
7. YAML agents (after fix)
8. Troubleshooting

---

## Before/After Comparison

### Before (README quickstart)

```bash
pip install praisonaiagents
export OPENAI_API_KEY="your-api-key"
```

### After (proposed replacement)

````markdown
## Quick Start with OpenAI (all platforms)

### 1. Install

**Python SDK only:**
```bash
pip install praisonaiagents
```

**CLI + YAML + full toolkit:**
```bash
pip install praisonai
```

### 2. Set your OpenAI API key

<Tabs>
  <Tab title="macOS / Linux">
    ```bash
    export OPENAI_API_KEY="sk-your-key-here"
    ```
  </Tab>
  <Tab title="Windows PowerShell">
    ```powershell
    $env:OPENAI_API_KEY="sk-your-key-here"
    ```
  </Tab>
  <Tab title="Windows CMD">
    ```cmd
    set OPENAI_API_KEY=sk-your-key-here
    ```
  </Tab>
  <Tab title=".env file (recommended)">
    Create a `.env` file in your project root:
    ```
    OPENAI_API_KEY=sk-your-key-here
    ```
    The CLI loads `.env` automatically. For Python SDK scripts, add:
    ```python
    from dotenv import load_dotenv
    load_dotenv()
    ```
  </Tab>
</Tabs>

### 3. Verify setup

```bash
praisonai doctor env --json
```

> **Windows note:** Use `--json` if text output shows encoding errors.

### 4. Run your first agent

```python
from praisonaiagents import Agent

agent = Agent(instructions="You are a helpful assistant.")
print(agent.start("Say hello in one sentence."))
```
````

### Before (multi-agent README)

```python
agents = Agents(agents=[research_agent, summarise_agent])
agents.start()
```

### After (proposed — interim until Issue #1 fixed)

````markdown
### Multi-agent teams

> **Note:** Requires praisonaiagents >= X.Y.Z (after streaming fix).

```python
from praisonaiagents import Agent, Agents

research = Agent(instructions="Research briefly")
writer = Agent(instructions="Summarize in one sentence")
result = Agents(agents=[research, writer]).start("What is Python?")
print(result)
```

If you see `Streaming is not supported in sync OpenAIAdapter`, upgrade praisonaiagents or use single-agent mode until fixed.
````

### Before (YAML README)

```bash
praisonai agents.yaml
```

### After (proposed — interim until Issue #2 fixed)

````markdown
### Run YAML agents

```bash
# Requires: pip install praisonai
praisonai agents.yaml --framework praisonai
```

Your `agents.yaml` must include:
```yaml
framework: praisonai
topic: "Your task here"
agents:
  ...
```
````

### Before (doctor example)

```bash
praisonai doctor --only python_version,openai_api_key
```

### After (proposed)

```bash
# Check environment and API keys
praisonai doctor env

# JSON output (recommended on Windows)
praisonai doctor env --json
```

---

## Screenshot Suggestions

1. **PowerShell env var** — screenshot of `$env:OPENAI_API_KEY` + successful `python -c "..."` agent run
2. **Doctor JSON output** — screenshot showing `"openai_api_key": "pass"`
3. **First agent output** — terminal showing agent response
4. **Windows vs macOS tabs** — side-by-side env var setup in Mintlify Tabs component

---

## OpenAI Onboarding Impact

| Gap | Onboarding impact |
|-----|-------------------|
| No Windows env vars | **Blocks step 2** on Windows (~50%+ of dev machines) |
| Package split unclear | Users install SDK, try YAML, get `praisonai: command not found` |
| No `.env` docs | Users re-export key every terminal session |
| Broken multi-agent docs | **Blocks step 3** — trust erosion after step 1 success |
| Broken YAML docs | **Blocks no-code path** entirely |
| Stale doctor docs | Verification step fails with `--only` error |

**Recommended priority:** Land code fixes (#1, #2) first, then update docs to remove interim workarounds.

---

# Approval Checklist

Use this when reviewing drafts before creating issues:

| # | Draft | Repo | Create? | Notes |
|---|-------|------|---------|-------|
| 1 | Multi-agent streaming | PraisonAI | ☐ | Critical — blocks README step 2 |
| 2 | YAML framework init | PraisonAI | ☐ | Critical — blocks YAML onboarding |
| 3 | Doctor Windows Unicode | PraisonAI | ☐ | Medium — workaround exists |
| 4 | UnboundLocalError print | PraisonAI | ☐ | Medium — error masking |
| 5 | simple_workflow.py | PraisonAI | ☐ | Medium — example fix |
| 6 | OpenAI onboarding docs | PraisonAIDocs | ☐ | High — after #1/#2 or with interim workarounds |

---

*Generated from OpenAI onboarding evaluation on 2026-06-10. Do not commit API keys. Rotate any keys used during testing.*
