> **Status:** Draft for manual review. Do NOT auto-create.
>
> **Repo test baseline:** PraisonAI main @ ce976671 | Windows 10 | Python 3.13.2 | praisonai 4.6.52 | praisonaiagents 1.6.52

# Issue Draft 4 — UnboundLocalError in YAML Failure Path

**Repository:** https://github.com/MervinPraison/PraisonAI

## 1. Issue Title

**Short title:** `UnboundLocalError: print` when YAML agent execution fails

**Alternative titles:**
- CLI masks YAML errors with secondary `print` shadowing bug
- `main()` local `from rich import print` breaks error reporting

**Recommended final title:** `CLI UnboundLocalError on print shadows real YAML execution errors`

---

## 2. Classification

- Bug
- Developer Experience

---

## 3. Severity

**Medium**

Primary failure (YAML/streaming) is the real issue, but this secondary crash hides the root error and produces a confusing stack trace.

---

## 4. Environment

```
OS:                 Windows 10
Python:             3.13.2
PraisonAI:          4.6.52
Installation:       editable from repo
```

---

## 5. Executive Summary

**What is broken:** When YAML agent execution fails, the CLI crashes with `UnboundLocalError: cannot access local variable 'print'` instead of showing the original error cleanly.

**Why it matters:** Debugging becomes harder — users see a Python scoping bug instead of the actionable LLM/framework error.

---

## 6. Architecture Analysis (Broken Flow)

```
praisonai agents.yaml --framework praisonai
        |
        v
  main() method
        |
        |  line 428 (elsewhere in main): from rich import print
        |  --> makes 'print' LOCAL for entire main() scope
        v
  generate_crew_and_kickoff() → fails (streaming error)
        |
        v
  line 860: print(result)
        |
        v
  *** UnboundLocalError: print not bound ***
        (backends branch with 'from rich import print' never ran)
```

---

## 7. Expected Architecture

```
main()
        |
        v
  execution fails with streaming error
        |
        v
  except handler OR print(result) using module-level print
        |
        v
  user sees original error message
```

---

## 8. Reproduction Steps

### Minimal Reproduction

```powershell
$env:OPENAI_API_KEY="sk-..."
praisonai _openai_test_agents.yaml --framework praisonai
```

(Any YAML that gets past init but fails during execution triggers this.)

---

## 9. Actual Result

```
Traceback (most recent call last):
  ...
  File "...\praisonai\cli\main.py", line 860, in main
    print(result)
    ^^^^^
UnboundLocalError: cannot access local variable 'print' where it is not associated with a value
```

---

## 10. Expected Result

Original execution error displayed. No secondary `UnboundLocalError`.

---

## 11. Root Cause Analysis

| File | Line | Issue |
|------|------|-------|
| `cli/main.py` | 71 | Module-level `from rich import print` |
| `cli/main.py` | 428 | `from rich import print` inside `main()` method |
| `cli/main.py` | 860 | `print(result)` — resolves to local `print` never assigned |

Python scoping: any `from rich import print` inside `main()` makes `print` local for the entire method body.

---

## 12. Impact Analysis

| Audience | Impact |
|----------|--------|
| YAML users | Medium — error messages obscured |
| New users | Medium — confusing stack traces |
| Windows users | Same |

---

## 13. Suggested Fix

### Quick Fix

Remove all `from rich import print` inside `main()`. Use module-level import only.

### Proper Fix

Use alias consistently:

```python
from rich import print as rprint
# use rprint everywhere in main()
```

### Long-Term Fix

Lint rule: no shadowing of builtins/module imports inside large methods.

---

## 14. Acceptance Criteria

- [ ] YAML execution failure shows original error, not UnboundLocalError
- [ ] `print(result)` at line 860 works on all code paths through `main()`
- [ ] No `from rich import print` inside `main()` method body

---

## 15. Regression Test Proposal

Unit test that mocks `generate_crew_and_kickoff` to raise `RuntimeError("test")` and asserts CLI surfaces that error without UnboundLocalError.

---

## 16. Additional Context

- **Triggered alongside:** Issue #1 (streaming failure during YAML execution)
- **Workaround:** None — fix the scoping issue directly

---
