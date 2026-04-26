# aimeet-live — AI Meet Live OpenClaw Skill

Connects your OpenClaw workspace to [AI Meet Live](https://aimeet.live). Your AI agent will silently extract meeting learnings at session start, brief your persona before calls, and retrieve past meeting content on demand. During live meetings, your persona can invoke tools from your OpenClaw workspace in real time.

---

## Install

### Download (during dev / before ClawHub)

1. Download [`aimeet-live.zip`](https://github.com/Belew-Consulting-LLC/aimeet-oc/releases/latest/download/aimeet-live.zip) from the latest release
2. Extract and copy the `aimeet-live/` folder into your OpenClaw skills directory:
   ```bash
   unzip aimeet-live.zip
   cp -r aimeet-live/ ~/.openclaw/skills/
   ```

### ClawHub (coming soon)

```bash
openclaw skills install aimeet-live
```

---

## Setup

1. **Get your API key** — AI Meet Live dashboard → Persona → OpenClaw tab → Generate API Key

2. **Add to MCP config:**
   ```json
   "mcp": {
     "servers": {
       "aimeet": {
         "transport": "sse",
         "url": "https://us.aimeet.live/api/openclaw/mcp",
         "headers": { "X-API-Key": "apm_YOUR_KEY_HERE" }
       }
     }
   }
   ```
   > Use your region URL — check your dashboard URL. `eu.aimeet.live` → use `https://eu.aimeet.live/api/openclaw/mcp`.

3. **Register your tools** — edit the `Tools to Register` section in `SKILL.md` with the tools from your workspace you want available during meetings.

4. **Restart OpenClaw:**
   ```bash
   openclaw gateway restart
   ```

---

## Docs

Full documentation: [aimeet.live/docs/openclaw](https://aimeet.live/docs/openclaw)
