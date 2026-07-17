---
name: denkops-mcp
description: Use when building or deploying an MCP server on DenkOps — defineMcp from @denkopsai/sdk/mcp (tools, resources, prompts), the stateless 2026-07-28 protocol the SDK serves for you, and connector:true for no-code OAuth.
---

# Building an MCP server on DenkOps

DenkOps hosts MCP servers on the same paved road as any API: you declare tools/resources/prompts,
the SDK serves the stateless MCP protocol, and `"connector": true` makes it OAuth-connectable to
Claude with no auth code. For a plain (non-MCP) HTTP API, use the `denkops-api` skill instead.

## What YOU write — two files (bun lane)

`index.ts` — **`export default`** a `defineMcp(...)` app. Import from the **`/mcp` subpath**:

```ts
import { defineMcp } from "@denkopsai/sdk/mcp";
import { z } from "zod";

export default defineMcp({
  name: "my-server",
  version: "1.0.0",
  tools: {
    add: {
      description: "Add two numbers.",
      input: z.object({ a: z.number(), b: z.number() }), // Zod (recommended) or a JSON Schema object
      handler: ({ a, b }) => ({ sum: a + b }),           // return a value; the SDK formats the result
    },
  },
  resources: {
    readme: { uri: "doc://readme", description: "About", read: () => ({ text: "hi" }) },
  },
  prompts: {
    summarize: {
      description: "Summarize text.",
      input: z.object({ text: z.string() }),
      build: ({ text }) => [{ role: "user", content: `Summarize: ${text}` }],
    },
  },
});
```

`package.json` — depend on `@denkopsai/sdk`, `hono`, and `zod`.

## What the SDK + platform give you — DO NOT write these yourself

- **The MCP protocol.** `defineMcp` serves a stateless `POST /mcp` (2026-07-28 RC): `tools/list`,
  `tools/call`, `resources/list`, `resources/read`, `prompts/list`, `prompts/get`. Do **not** hand-roll
  JSON-RPC. Stateless means no session pinning — it scales across nodes.
- **Auth, health, tracing, timeout** — from the paved-road wrapper, same as any DenkOps app.
- **OAuth** — set `"connector": true` in `denkops.json` and deploy. DenkOps hosts the whole handshake
  (dynamic client registration + PKCE) via `connect.denkops.com` and serves the
  `/.well-known/oauth-protected-resource` discovery document. You write no auth code.

## Connect it to Claude

1. Set `"connector": true` in `denkops.json` and **"deploy on DenkOps"**.
2. In Claude, add a custom connector with the URL `https://<slug>.<app-domain>/mcp` and authorize.
3. **Access control** (dashboard Connect page): **Public**, an **email allowlist**, or the default
   **project members only**. People authorize with GitHub/Google/Microsoft; revoke any connection there.

## Schema notes

- `input` accepts a **Zod** schema (recommended: runtime validation + types + the JSON Schema shown in
  `tools/list`) or a **plain JSON Schema** object (no runtime validation). Omit `input` for no arguments.

## Out of scope (today)

MCP Apps (sandboxed UIs), Tasks, and server-initiated elicitation are not part of the builder yet.
Python is not supported yet — use the bun lane.

## Deploy

Say **"deploy on DenkOps"**. Your MCP goes live at `https://<slug>.<app-domain>/mcp`; the deploy
result includes the `project_id`. Editing `connector`/tools then redeploying applies on the next deploy.

