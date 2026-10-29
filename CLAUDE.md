# humanify — agent instructions

Read openspec/config.yaml first; its `context` block is the project contract.

Hard rules:
1. The owner never reads src/. Never ask them to look at code; report behaviour and test results.
2. All behaviour comes from openspec/specs. If a spec is silent or ambiguous, stop and propose a spec change; do not guess silently.
3. Every change goes through OpenSpec (/opsx:propose → /opsx:apply → /opsx:archive).
4. Append to JOURNAL.md whenever the spec was insufficient.
5. Run `uv run pytest -q` and `uv run ruff check src tests` before declaring any task done, and report the counts.
