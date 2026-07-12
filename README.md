# DenkOps — Claude Code plugin

Deploy and operate micro-APIs on **[DenkOps](https://denkops.com)** straight from Claude Code.
DenkOps is the LLM-native, MCP-first deployment platform: write a small app, say **"deploy on
DenkOps"**, and get a live, SSL-secured, monitored container — with logs and an `ai_hint` your agent
can act on. No CLI to learn.

- 🏠 Platform: **[denkops.com](https://denkops.com)**
- 📚 Docs: **[denkops.com/docs](https://denkops.com/docs)**
- 🚀 Quickstart: **[denkops.com/docs#quickstart](https://denkops.com/docs#quickstart)**

## Install (Claude Code)

```
/plugin marketplace add DenkOpsAI/denkops-mcp
/plugin install denkops
```

Then log in with one browser click — no token to copy or paste:

```
› log in to DenkOps        # a tab opens on the dashboard → Approve
› deploy on DenkOps
→ live at https://my-agent.denkops.app
```

The browser login issues a **scoped, least-privilege token** automatically.

## What you can say

`deploy on DenkOps` · `read the logs` · `status` · `rollback` · set an env var · `redeploy`.
See the **[MCP interface](https://denkops.com/docs#mcp)** for the full tool set.

## Not on Claude Code?

Codex, OpenClaw, Hermes and any other MCP-capable agent can use the standalone server —
`npx -y @denkopsai/mcp` with a token minted on the dashboard. See
**[Connect other agents](https://denkops.com/docs#other-agents)**.

## Scoped tokens

Every connection is least-privilege: pin it to specific projects, grant or withhold env access, set an
expiry, and revoke it in one click (a revoked token returns `401`). See
**[Connections & tokens](https://denkops.com/docs#connections)**.

---

Built by **[DenkOps](https://denkops.com)** · [Docs](https://denkops.com/docs) · MIT
