> **Status:** Draft for manual review. Do NOT auto-create.
>
> **Repo test baseline:** PraisonAI main @ ce976671 | Windows 10 | Python 3.13.2 | praisonai 4.6.52 | praisonaiagents 1.6.52

# Issue Draft 2 — YAML Framework Initialization Bug

**Repository:** https://github.com/MervinPraison/PraisonAI

## 1. Issue Title

**Short title:** YAML CLI fails: framework adapter initialized before YAML is loaded

**Alternative titles:**
- `praisonai agents.yaml` crashes with `Unknown framework_adapters plugin: ''`
- `AgentsGenerator.__init__` requires framework before reading YAML `framework:` key
- README YAML quickstart broken without `--framework praisonai`

**Recommended final title:** `YAML agents fail on CLI: framework adapter created in __init__ before YAML framework is read`

---

## 2. Classification

- Bug
- Regression
- Developer Experience

---

## 3. Severity

**Critical**

README documents `praisonai agents.yaml` as the no-code onboarding path. It fails before YAML is parsed unless `--framework praisonai` is passed on the CLI.

---

## 4. Environment

```
OS:                 Windows 10 (AMD64)
Python:             3.13.2
PraisonAI:          4.6.52
PraisonAIAgents:    1.6.52
Installation:       editable from repo (main @ ce976671)
LLM provider:       OpenAI
Model:              gpt-4o-mini
```

---

## 5. Executive Summary

**What is broken:** Running `praisonai agents.yaml` (as documented in README) crashes immediately with `Unknown praisonai.framework_adapters plugin: ''`.

**Why it matters:** YAML is marketed as the "no code" path. New users following README cannot run their first YAML workflow.

**Impact on new users:** Blocks the documented alternative to Python API after install + API key.

**Impact on OpenAI onboarding:** Even with `OPENAI_API_KEY` set, YAML path never reaches the LLM.

---

## 6. Architecture Analysis (Broken Flow)

```
User: praisonai agents.yaml
        |
        v
  CLI main.py (framework="" from constructor)
        |
        v
  AgentsGenerator(agent_file, framework="", ...)   <-- empty framework
        |
        v
  __init__ line 239:
  self.framework_adapter = _get_framework_adapter("")
        |
        v
  _registry.create("")
        |
        v
  *** ValueError: Unknown plugin: '' ***
        |
        X  (YAML never loaded — crash in __init__)
```

YAML `framework: praisonai` is only read later in `generate_crew_and_kickoff()` at line 596 — too late.

---

## 7. Expected Architecture

```
User: praisonai agents.yaml
        |
        v
  AgentsGenerator(agent_file, framework="", ...)
        |
        |  __init__: NO adapter creation yet
        v
  generate_crew_and_kickoff()
        |
        v
  Load YAML → config.get('framework', 'praisonai')
        |
        v
  _get_framework_adapter('praisonai')
        |
        v
  adapter.run(config, ...) → OpenAI → success
```

---

## 8. Reproduction Steps

### Minimal Reproduction

Create `agents.yaml`:

```yaml
framework: praisonai
topic: "Reply with exactly: YAML_OK"

agents:
  assistant:
    role: Assistant
    goal: Answer briefly
    instructions: "Reply with exactly the requested text."
```

Run:

```bash
export OPENAI_API_KEY=sk-...
praisonai agents.yaml
```

### Full Reproduction

```bash
praisonai examples/cookbooks/yaml/news_monitor_scheduled.yaml
```

### CLI Workaround (partial)

```bash
praisonai agents.yaml --framework praisonai
```

This passes init but still hits Issue #1 (multi-agent streaming) during execution.

---

## 9. Actual Result

```
Traceback (most recent call last):
  File "...\praisonai\__main__.py", line 186, in main
    _run_legacy(argv)
  File "...\praisonai\__main__.py", line 132, in _run_legacy
    result = praison.main()
  File "...\praisonai\cli\main.py", line 851, in main
    agents_generator = AgentsGenerator(
        self.agent_file,
        self.framework,
        ...
    )
  File "...\praisonai\agents_generator.py", line 239, in __init__
    self.framework_adapter = self._get_framework_adapter(framework)
  File "...\praisonai\agents_generator.py", line 254, in _get_framework_adapter
    return self._adapter_registry.create(framework)
  File "...\praisonai\_registry.py", line 159, in resolve
    raise ValueError(
ValueError: Unknown praisonai.framework_adapters plugin: ''. Available: ['ag2', 'autogen', 'autogen_v4', 'crewai', 'praisonai']
```

---

## 10. Expected Result

- `praisonai agents.yaml` reads `framework: praisonai` from YAML
- No `--framework` CLI flag required when YAML specifies framework
- Execution proceeds to agent kickoff

---

## 11. Root Cause Analysis

| File | Line | Issue |
|------|------|-------|
| `cli/main.py` | 851-858 | Passes `self.framework` (empty string) to constructor |
| `agents_generator.py` | 239 | Creates adapter in `__init__` |
| `agents_generator.py` | 596 | Reads framework from YAML — **after** init |
| `PraisonAI.__init__` | 267 | `framework=""` default |

Recent wrapper refactor (#1797, #1855) likely introduced eager adapter creation in `__init__`.

---

## 12. Impact Analysis

| Audience | Impact |
|----------|--------|
| New users | Critical — README YAML path broken |
| OpenAI users | High — cannot use documented no-code path |
| YAML users | Critical |
| Windows users | Same failure (not OS-specific) |

---

## 13. Suggested Fix

### Quick Fix

Default CLI framework to `praisonai` when agent file ends with `.yaml`:

```python
# cli/main.py
if args.command.endswith('.yaml') and not self.framework:
    self.framework = 'praisonai'
```

### Proper Fix

Defer adapter creation to `generate_crew_and_kickoff()`:

```python
# agents_generator.py __init__
self.framework_adapter = None  # lazy

def generate_crew_and_kickoff(self):
    config = self._load_config()
    framework = self.framework or config.get('framework', 'praisonai')
    self.framework_adapter = self._get_framework_adapter(framework)
    ...
```

### Long-Term Fix

- Change YAML default framework from `crewai` to `praisonai` (line 596)
- Add CLI validation: if YAML file provided, never require `--framework` when YAML has `framework:`

---

## 14. Acceptance Criteria

- [ ] `praisonai agents.yaml` works with `framework: praisonai` in YAML, no CLI flag
- [ ] README YAML example runs successfully (after Issue #1 fix)
- [ ] `--framework` CLI flag still overrides YAML when explicitly set
- [ ] Regression test: YAML init without CLI framework arg

---

## 15. Regression Test Proposal

```python
def test_yaml_agents_no_cli_framework(tmp_path, monkeypatch):
    yaml_file = tmp_path / "agents.yaml"
    yaml_file.write_text("framework: praisonai\ntopic: hi\nagents:\n  a:\n    role: A\n    goal: g\n    instructions: i\n")
    gen = AgentsGenerator(str(yaml_file), framework="", config_list=[{"model": "gpt-4o-mini"}])
    assert gen.framework_adapter is None or gen.framework_adapter.name == "praisonai"
    # should not raise in __init__
```

---

## 16. Additional Context

- **Related:** [#1652](https://github.com/MervinPraison/PraisonAI/issues/1652) — broader adapter migration; this is a specific init-order bug
- **Related:** [#1738](https://github.com/MervinPraison/PraisonAI/issues/1738) (closed) — YAML surface gaps
- **Workaround:** `praisonai agents.yaml --framework praisonai`
- **Default when YAML omits framework:** `crewai` (line 596) — wrong for OpenAI-first onboarding

---
