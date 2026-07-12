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

## Config & deploy

- `denkops.json` = `{ name, slug, runtime }`. It's created automatically on the first deploy
  (inferred from the directory); edit `slug`/`name` to pin them.
- To ship: say **"deploy on DenkOps"**. The MCP packs the current directory and deploys it; a new
  project is created zero-config. Your app goes live at `https://<slug>.<app-domain>`, and the deploy
  result includes the `project_id` and the `DENKOPS_API_KEY` to call it.

