> **Status:** Draft for manual review. Do NOT auto-create.
>
> **Repo test baseline:** PraisonAI main @ ce976671 | Windows 10 | Python 3.13.2 | praisonai 4.6.52 | praisonaiagents 1.6.52

# Issue Draft 3 — Windows Doctor Env Unicode Crash

**Repository:** https://github.com/MervinPraison/PraisonAI

## 1. Issue Title

**Short title:** `praisonai doctor env` crashes on Windows with charmap UnicodeEncodeError

**Alternative titles:**
- Doctor text formatter Unicode symbols fail on cp1252 console
- Windows health check unusable without `--json` flag

**Recommended final title:** `doctor env crashes on Windows: charmap codec cannot encode Unicode status symbols`

---

## 2. Classification

- Bug
- Compatibility
- Developer Experience

---

## 3. Severity

**Medium**

Doctor is a diagnostic tool, not the primary execution path. Workaround exists (`--json`, `PYTHONIOENCODING=utf-8`). But it blocks the recommended "verify your setup" step for Windows users during OpenAI onboarding.

---

## 4. Environment

```
OS:                 Windows 10 (AMD64), console encoding cp1252
Python:             3.13.2
PraisonAI:          4.6.52
PraisonAIAgents:    1.6.52
Installation:       editable from repo
LLM provider:       OpenAI (OPENAI_API_KEY set)
```

---

## 5. Executive Summary

**What is broken:** `praisonai doctor env` fails on Windows default console with a charmap encoding error.

**Why it matters:** Doctor is the first diagnostic command users run when setup fails. On Windows, it crashes instead of reporting status.

**Impact on OpenAI onboarding:** Users cannot verify `OPENAI_API_KEY` via the documented human-readable doctor output.

---

## 6. Architecture Analysis (Broken Flow)

```
User: praisonai doctor env
        |
        v
  DoctorHandler → DoctorEngine → env checks (PASS)
        |
        v
  TextFormatter.format_report()
        |
        |  STATUS_SYMBOLS: ✓ ⚠ ✗ ○  (Unicode)
        v
  sys.stdout.write(formatted_text)
        |
        v
  Windows cp1252 encoder
        |
        v
  *** UnicodeEncodeError: 'charmap' codec can't encode ***
```

---

## 7. Expected Architecture

```
User: praisonai doctor env
        |
        v
  DoctorEngine → checks complete
        |
        v
  TextFormatter (encoding-safe)
        |
        +--> detect stdout encoding
        |         |
        |         +--> if not UTF-8: use ASCII symbols [OK] [WARN] [FAIL]
        v
  sys.stdout.write() → success on all platforms
```

---

## 8. Reproduction Steps

### Minimal Reproduction

```powershell
$env:OPENAI_API_KEY="sk-..."
praisonai doctor env
```

### Full Reproduction

Default Windows PowerShell / cmd.exe, no `PYTHONIOENCODING` set.

### Workaround

```powershell
$env:PYTHONIOENCODING="utf-8"
praisonai doctor env
# OR
praisonai doctor env --json
```

---

## 9. Actual Result

```
ERROR: Doctor error: 'charmap' codec can't encode characters in position 25-94: character maps to <undefined>
```

With `--json`: exits 0, all checks pass including `openai_api_key`.

---

## 10. Expected Result

Human-readable doctor report on Windows showing:

```
✓ Python Version: Python 3.13.2 (>= 3.9 required)
✓ OpenAI API Key: OPENAI_API_KEY configured
...
```

Or ASCII-safe equivalents on non-UTF-8 consoles.

---

## 11. Root Cause Analysis

| File | Line | Issue |
|------|------|-------|
| `cli/features/doctor/formatters.py` | 168-174 | `STATUS_SYMBOLS` uses Unicode ✓ ⚠ ✗ ○ |
| `cli/features/doctor/formatters.py` | 136-141 | Writes directly to `sys.stdout` without encoding guard |
| Windows default | — | Console encoding `cp1252` cannot encode U+2713, U+26A0, etc. |

Gateway had the same class of bug — fixed in PR #1754 via `unicode_utils.py`. Doctor formatter was not updated.

---

## 12. Impact Analysis

| Audience | Impact |
|----------|--------|
| Windows users | High — primary diagnostic command fails |
| New users | Medium — cannot verify API key setup easily |
| OpenAI users | Medium — `openai_api_key` check exists but unreachable in text mode |
| CI/Linux users | None |

---

## 13. Suggested Fix

### Quick Fix

Use ASCII symbols when encoding is not UTF-8:

```python
STATUS_SYMBOLS_ASCII = {PASS: "[OK]", WARN: "[!]", FAIL: "[X]", SKIP: "[-]"}
```

### Proper Fix

Reuse `gateway/unicode_utils.py` `safe_error_message()` pattern or centralize in shared `console_utils.py`.

Force UTF-8 stdout on Windows at CLI entry (with fallback):

```python
if sys.platform == "win32":
    sys.stdout.reconfigure(encoding="utf-8", errors="replace")
```

### Long-Term Fix

Apply encoding-safe output to all Rich/CLI formatters project-wide.

---

## 14. Acceptance Criteria

- [ ] `praisonai doctor env` succeeds on Windows 10 default console
- [ ] `praisonai doctor env --json` still works
- [ ] Unicode symbols shown when terminal supports UTF-8
- [ ] ASCII fallback on cp1252
- [ ] Unit test for formatter with mock cp1252 stdout

---

## 15. Regression Test Proposal

```python
def test_text_formatter_cp1252_safe():
    formatter = TextFormatter(no_color=True)
    report = make_sample_report()
    # Mock stdout with cp1252 encoding
    assert formatter.format_report(report).encode("cp1252", errors="strict")
```

---

## 16. Additional Context

- **Related (not duplicate):** [PraisonAIDocs #446](https://github.com/MervinPraison/PraisonAIDocs/issues/446) — Gateway Telegram charmap fix only
- **Related:** PraisonAI #1752 / PR #1754 — `unicode_utils.py` pattern exists, not reused here
- **Workaround:** `--json` or `PYTHONIOENCODING=utf-8`

---
