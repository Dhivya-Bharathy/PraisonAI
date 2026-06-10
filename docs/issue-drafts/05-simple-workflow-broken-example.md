> **Status:** Draft for manual review. Do NOT auto-create.
>
> **Repo test baseline:** PraisonAI main @ ce976671 | Windows 10 | Python 3.13.2 | praisonai 4.6.52 | praisonaiagents 1.6.52

# Issue Draft 5 — simple_workflow.py Broken Example

**Repository:** https://github.com/MervinPraison/PraisonAI

## 1. Issue Title

**Short title:** `simple_workflow.py` NameError: `AgentFlow` not defined

**Alternative titles:**
- Workflow example imports `Workflow` but uses `AgentFlow`
- README-linked workflow example fails at import time

**Recommended final title:** `examples/python/workflows/simple_workflow.py: NameError AgentFlow not imported`

---

## 2. Classification

- Bug
- Documentation (example code)

---

## 3. Severity

**Medium**

Example is linked from README workflows table. Fails before any LLM call — pure code error.

---

## 4. Environment

```
OS:                 Windows 10
Python:             3.13.2
PraisonAIAgents:    1.6.52
LLM provider:       OpenAI (never reached)
```

---

## 5. Executive Summary

**What is broken:** README-linked `simple_workflow.py` crashes with `NameError` on line 26.

**Why it matters:** Listed as the primary "Simple Workflow" example in README feature table.

---

## 6. Architecture Analysis (Broken Flow)

```
User runs simple_workflow.py
        |
        v
  from praisonaiagents import Agent, Workflow   <-- imports Workflow
        |
        v
  workflow = AgentFlow(...)                     <-- NameError
        X
```

---

## 7. Expected Architecture

```
User runs simple_workflow.py
        |
        v
  from praisonaiagents import Agent, AgentFlow
        |
        v
  workflow = AgentFlow(steps=[researcher, writer])
        |
        v
  workflow.start("...") → OpenAI agents execute
```

---

## 8. Reproduction Steps

### Minimal Reproduction

```bash
python examples/python/workflows/simple_workflow.py
```

---

## 9. Actual Result

```
Traceback (most recent call last):
  File "examples/python/workflows/simple_workflow.py", line 26, in <module>
    workflow = AgentFlow(
               ^^^^^^^^^
NameError: name 'AgentFlow' is not defined
```

---

## 10. Expected Result

Workflow runs two agents sequentially and prints final output.

---

## 11. Root Cause Analysis

| File | Line | Issue |
|------|------|-------|
| `examples/python/workflows/simple_workflow.py` | 8 | Imports `Workflow`, not `AgentFlow` |
| `examples/python/workflows/simple_workflow.py` | 26 | Uses `AgentFlow` |

Likely rename oversight during API consolidation (#1855).

---

## 12. Impact Analysis

| Audience | Impact |
|----------|--------|
| New users following README | Medium |
| OpenAI users | Low until fix — then depends on Issue #1 for multi-step |
| Documentation readers | Misleading link |

---

## 13. Suggested Fix

### Quick Fix

```python
from praisonaiagents import Agent, AgentFlow
```

### Proper Fix

Add example to CI smoke test suite; run `python examples/python/workflows/simple_workflow.py` in example validation.

---

## 14. Acceptance Criteria

- [ ] `python examples/python/workflows/simple_workflow.py` runs without NameError
- [ ] README link target works
- [ ] Example validation script includes this file

---

## 15. Regression Test Proposal

Example audit runner: import/execute check for all README-linked examples.

---

## 16. Additional Context

- **README reference:** `examples/python/workflows/simple_workflow.py` in Workflows feature table
- **Related PR context:** #1855 consolidated-params API alignment

---
