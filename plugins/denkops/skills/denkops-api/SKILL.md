---
name: denkops-api
description: Use when building or deploying a DenkOps API/app (bun + Hono or python + FastAPI) — the paved-road conventions: what files to write, what the wrapper provides (/health, auth, ai_hint, timeout), the @denkopsai/sdk key-value store on /persist, DENKOPS_API_KEY, and how to deploy.
---

# Building a DenkOps API

DenkOps runs your API on a "paved road": you write a small app; the platform wraps it with auth,
health, tracing, timeouts, and TLS, and serves it. Follow these conventions.

## Two lanes (set by `runtime` in denkops.json)

- **bun** (default, ~80% of APIs): bun + Hono.
- **python** (AI/ML / heavier data workloads): python + FastAPI.

## What YOU write

**bun lane** — two files:
- `index.ts` — **`export default`** a Hono app.
- `package.json` — depend on `hono`.

```ts
// index.ts
import { Hono } from "hono";
const app = new Hono();
app.get("/hello", (c) => c.json({ message: "hi" }));
export default app; // DenkOps wraps and serves this
```

**python lane** — two files:
- `main.py` — a FastAPI app named **`app`** (imported as `main:app`).
- `requirements.txt` — list `fastapi` (uvicorn is provided by the wrapper).

```python
# main.py
from fastapi import FastAPI
app = FastAPI()

@app.get("/hello")
def hello():
    return {"message": "hi"}
```

## What the wrapper gives you for free — DO NOT write these yourself

- **`/health`** — public, returns `{"status":"ok"}`. Don't define your own.
- **Auth-by-Default** — every route except `/health` requires `Authorization: Bearer $DENKOPS_API_KEY`.
  Missing/wrong key → 401 with the `ai_hint` JSON error contract. Don't add your own auth middleware.
- **Tracing** and a request **timeout** (`DENKOPS_TIMEOUT_SEC`).
- **DO NOT** call `Bun.serve` / `uvicorn.run` or bind a port — the wrapper serves your app on `:8080`.
  Just export the app.

## Persistent storage

Small, durable key-value storage is provided by the DenkOps SDK — a SQLite store on the per-project
**`/persist`** volume (survives redeploys).

**bun** — `bun add @denkopsai/sdk`:
```ts
import denkops from "@denkopsai/sdk";
denkops.store.set("visits", "1");        // string values; optional { ttlSeconds }
const v = denkops.store.get("visits");   // string | null
```

**python** — add `denkops` to `requirements.txt`:
```python
import denkops
denkops.store.set("visits", "1")
visits = denkops.store.get("visits")     # str | None
```

For raw files, write under `/persist` directly — its path is also in `DENKOPS_PERSIST`.

## Environment

- **`DENKOPS_API_KEY`** — the bearer key clients (and the wrapper) use.
- `DENKOPS_TIMEOUT_SEC` — the per-request timeout (managed by the platform).
- Put project config/secrets in a `.env` file (deployed with the project); don't hard-code secrets.

## Limits

- Default **128 MB** RAM per app — keep dependencies lean.

## Custom connectors for Claude

A deployed DenkOps app can be added to Claude as a custom connector. Two ways, from simplest to
most seamless:

- **Simple — static header.** Add your app in Claude as a custom connector with URL
  `https://<slug>.<app-domain>` and header `Authorization: Bearer <DENKOPS_API_KEY>` (the
  `DENKOPS_API_KEY` from your deploy result). This works today with zero config, since every
  deployed app is already bearer-gated. On Claude's side this is an org-admin-entered, static-header
  connector (currently in beta there).
- **OAuth — no code required.** Set `"connector": true` in `denkops.json` and deploy. In Claude, add
  a custom connector with URL `https://<slug>.<app-domain>/mcp` — DenkOps hosts the whole OAuth
  handshake (dynamic client registration + PKCE) via the `connect.denkops.com` authorization server,
  and serves `/.well-known/oauth-protected-resource` at your app's origin so Claude can discover it.
  You don't write any auth code.

**Access control** (OAuth path) is set by the project owner on the dashboard's Connect page:
**Public** (anyone who reaches the URL), an **email allowlist** (only those exact emails), or the
default — project members only. People authorizing a connection sign in with GitHub, Google, or
Microsoft; each resulting connection is listed on the Connect page and can be revoked there, taking
effect immediately.

## Config & deploy

- `denkops.json` = `{ name, slug, runtime }`, plus optional `region`, `streaming`, and `connector`. It's
  created automatically on the first deploy (inferred from the directory); edit `slug`/`name` to pin them.
- `"connector": true` makes the app connectable to Claude as a custom connector over OAuth — see
  "Custom connectors for Claude" above. Applies on the next deploy.
- `"streaming": true` opts the project out of response buffering so it can stream (SSE / chunked /
  long-poll) — set it for LLM token streams, progress events, etc. Trade-off: a streaming project has
  no response-size cap. Applies on the next deploy (say **"deploy on DenkOps"** to apply a change).
- To ship: say **"deploy on DenkOps"**. The MCP packs the current directory and deploys it; a new
  project is created zero-config. Your app goes live at `https://<slug>.<app-domain>`, and the deploy
  result includes the `project_id` and the `DENKOPS_API_KEY` to call it.

