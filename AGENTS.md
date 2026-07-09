# AGENTS.md

Guidance for future coding agents working in this repository.

## Project purpose

This repo contains `OPC`, a Bash wrapper with embedded Node.js that reports OpenCode Go usage more completely than the web UI.

It:

- loads local config from `.env`
- authenticates to OpenCode using the `auth` cookie
- discovers OpenCode's internal `usage.list` server function from the web app assets
- fetches paginated usage records
- incrementally merges new records into a local cache
- parses OpenCode Go quota/reset windows
- prints request/cost/token summaries by period, model, and optionally day

## Important files

- `OPC` — main executable script
- `.env.example` — config template; safe to commit
- `.env` — real auth config; **must not be committed**
- `data/usage.json` — local merged usage cache; **must not be committed**
- `data/meta.json` — local cache metadata; **must not be committed**
- `README.md` — user documentation

## Security rules

Never commit or print secrets.

Ignored sensitive/local paths:

```gitignore
.env
data/
```

`OPENCODE_AUTH` is an auth cookie and should be treated as a secret.

## Development notes

- Keep the tool self-contained: avoid adding dependencies unless necessary.
- The script should continue to run as `./OPC` with only Node.js available.
- Relative cache paths should resolve next to the `OPC` script, not the caller's current working directory.
- Page snapshots should remain disabled by default. Only write `data/pages/` when `--save-pages` is explicitly passed.
- Preserve the 5 sequential request fingerprint overlap logic for incremental cache updates.

The overlap fingerprint intentionally includes more than the OpenCode `id`:

- request id
- workspace id
- timestamp
- model/provider
- input/output/reasoning/cache token counts
- cost
- key id/session id
- plan

## Verification

Useful checks:

```bash
./OPC --help
./OPC --json
./OPC --start 01-01-2026 --end 31-01-2026
```

For a non-destructive cache check, run:

```bash
./OPC
```

A healthy cached run should usually fetch only the first API page if no new usage occurred.

Use full refresh only when needed:

```bash
./OPC --refresh
```
