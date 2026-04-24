---
name: vrin-status
description: Show whether the Vrin CLI is signed in and reachable from this machine. Use only when the user explicitly asks (e.g. "/vrin-status", "is vrin working?", "am I signed into vrin?").
metadata:
  author: Vrin
  version: 0.1.0
  category: diagnostics
  tags: [vrin, diagnostics, auth]
allowed-tools: Bash(vrin whoami:*), Bash(vrin health:*), Bash(vrin --version:*)
user-invocable: true
disable-model-invocation: true
---

# vrin-status

User-only diagnostic. Prints who the CLI is signed in as and confirms the backend is reachable. Not auto-fired — only runs when the user invokes `/vrin-status` or asks directly.

## How to run

Run these three commands via Bash and summarize the results:

```
vrin --version
vrin whoami --json
vrin health --json
```

## Parsing

- `vrin --version` → one line like `vrin 1.3.2`. Report the version.
- `vrin whoami --json` →
  - If `{"ok": true, "data": {"source": "credentials_file", "email": "…", ...}}` — signed in via `vrin login`. Report the email.
  - If `{"ok": true, "data": {"source": "env", ...}}` — signed in via `VRIN_API_KEY` env var. Report that.
  - If `{"ok": false, "error": {"code": "not_signed_in", ...}}` — **not signed in**. Tell the user to run `vrin login`.
- `vrin health --json` → `{"ok": true, "data": {"status": "healthy", ...}}` means the backend is reachable. If not, report the error.

## Output shape

Give the user a crisp 3-line status:

```
✓ vrin 1.3.2
✓ signed in as alex@acme.com (credentials file)
✓ backend healthy
```

If any line fails, show ✗ and the remedy. Do not dump raw JSON unless the user asks.

## Scope

This skill does **not** show query history, last retrieval details, or usage metrics — those belong on a future `/vrin-history` skill. Stay in the auth/health lane.
