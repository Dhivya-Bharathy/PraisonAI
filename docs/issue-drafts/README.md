# OpenAI Onboarding — Issue Draft Index

Evaluated against PraisonAI `main` @ `ce976671` (2026-06-08).

| # | File | Repository | Status | GitHub |
|---|------|------------|--------|--------|
| 1 | [01-multi-agent-openai-streaming-failure.md](01-multi-agent-openai-streaming-failure.md) | PraisonAI | **Duplicate — do not file** | Already reported: [#1733](https://github.com/MervinPraison/PraisonAI/issues/1733), [#1730](https://github.com/MervinPraison/PraisonAI/issues/1730) (closed, still reproduces on 4.6.52) |
| 2 | [02-yaml-framework-init-bug.md](02-yaml-framework-init-bug.md) | PraisonAI | **Filed** | [#1877](https://github.com/MervinPraison/PraisonAI/issues/1877) |
| 3 | [03-doctor-env-windows-unicode-crash.md](03-doctor-env-windows-unicode-crash.md) | PraisonAI | **Filed** | [#1878](https://github.com/MervinPraison/PraisonAI/issues/1878) |
| 4 | [04-unboundlocalerror-yaml-print-shadow.md](04-unboundlocalerror-yaml-print-shadow.md) | PraisonAI | **Filed** | [#1879](https://github.com/MervinPraison/PraisonAI/issues/1879) |
| 5 | [05-simple-workflow-broken-example.md](05-simple-workflow-broken-example.md) | PraisonAI | **Filed** | [#1880](https://github.com/MervinPraison/PraisonAI/issues/1880) |
| 6 | [06-openai-onboarding-docs-gaps.md](06-openai-onboarding-docs-gaps.md) | PraisonAIDocs | **Filed** | [#504](https://github.com/MervinPraison/PraisonAIDocs/issues/504) |

## Duplicate search notes

- **#1:** Exact match — same error (`Streaming is not supported in sync OpenAIAdapter`), same README repro code. Reopen [#1733](https://github.com/MervinPraison/PraisonAI/issues/1733) instead of filing anew.
- **#2:** Related [#1652](https://github.com/MervinPraison/PraisonAI/issues/1652) (adapter migration) but not the same init-order bug.
- **#3:** Related [PraisonAIDocs #446](https://github.com/MervinPraison/PraisonAIDocs/issues/446) (Gateway Telegram charmap only).
- **#4–#5:** No existing issues found.
- **#6:** Related closed onboarding docs issues; none cover Windows env vars + broken multi-agent/YAML paths.
