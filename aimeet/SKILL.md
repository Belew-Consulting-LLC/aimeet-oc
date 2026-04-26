---
name: aimeet
description: "Connects your workspace to AI Meet Live. Check for unlearned meetings silently at every session start and extract them automatically. When the user mentions an upcoming meeting, sync workspace context to their AI persona. When they mention a call just ended, offer to extract learnings. When asked about a past meeting, retrieve it directly. All MCP tools are available via the aimeet MCP server — do not announce tool calls, just do them."
emoji: "🎙️"
homepage: https://aimeet.live/docs/openclaw
metadata:
  openclaw:
    always: true
    requires:
      config:
        - mcp.servers.aimeet
    optional:
      - chief-of-staff
---

## Setup (one-time)

**1. Get your API key and region URL** from the AI Meet Live dashboard:
**Persona → OpenClaw tab → Generate API Key**

> ⚠️ Keys are region-scoped. If your dashboard is at `eu.aimeet.live`, use that URL — not `us.aimeet.live`.

**2. Add AI Meet to your MCP config:**

```json
"mcp": {
  "servers": {
    "aimeet": {
      "transport": "sse",
      "url": "https://us.aimeet.live/api/openclaw/mcp",
      "headers": {
        "X-API-Key": "apm_YOUR_KEY_HERE"
      }
    }
  }
}
```

**3. Restart OpenClaw:**

```bash
openclaw gateway restart
```

**4. Verify the connection** by calling `aimeet_status`. If it returns connection status and an unlearned meeting count, setup is working. If it fails, check:
- API key starts with `apm_`
- Region URL matches your dashboard URL
- MCP entry is under `mcp.servers.aimeet` (exact key name)

---

## Access Control (configure in persona settings)

Before live tool calling works during meetings, configure two settings in the AI Meet Live dashboard under **Persona → OpenClaw tab → Live Tools**:

**Speaker gate** — who can trigger tool calls:
- `authorized_only` (default) — only speakers on the persona's Authorized Speakers list can invoke tools
- `everyone` — any meeting participant's utterance can trigger tools

**Trigger phrase** — optional keyword that must appear in the utterance. If set to `"hey beau"`, the participant must say something like *"Hey Beau, look up Acme"* — utterances without the phrase won't trigger tools. Leave blank to allow any utterance (subject to the speaker gate).

---

## Session Start Behavior (do this silently every time)

At the start of every session, without announcing it:

1. Call `aimeet_status`
2. If `unlearned_meetings > 0`: call `aimeet_learn_all` immediately and write results to workspace files (see Workspace Write-Back below). Do not narrate — just do it.
3. Call `aimeet_register_tools` with tools from the **Tools to Register** section below.
4. If `aimeet_upcoming` returns meetings within the next 60 minutes: surface them naturally once — e.g. *"You have a call with Sarah in 40 minutes — want me to brief [persona name]?"*

Do not call `aimeet_learn_all` more than once per session start — results are cached.

### Periodic Sweeps

If this agent runs periodic sweeps via cron or heartbeat, check `aimeet_status` for new `unlearned_meetings` on each cycle — not just session start. Meetings that end mid-session won't be caught otherwise. If new unlearned meetings appear, call `aimeet_learn_meeting` on the specific meeting and write results back.

---

## Error Handling

If any aimeet tool call fails:

- **On session start:** Skip silently and continue. Do not block the session or retry repeatedly. The agent should work normally without AI Meet — it just won't have meeting data this session.
- **On user-triggered action** (e.g., "pull up that meeting"): Tell the user briefly — *"AI Meet isn't responding right now. I'll try again next session, or you can check the dashboard directly."*
- **Never retry more than once** in the same turn. If the retry fails, move on.
- **Never surface MCP errors unprompted.** If the user didn't ask about meetings, they don't need to know AI Meet is down.

---

## Tools to Register

Edit this list to include the tools from your OpenClaw workspace you want the AI Meet persona to invoke during live meetings. The agent reads these at every session start and calls `aimeet_register_tools` silently. Registration is idempotent — safe to run every session.

```json
[
  {
    "name": "your_tool_name",
    "description": "What this tool does and when the persona should call it. The LLM reads this to decide whether to invoke it during a meeting.",
    "display_name": "Your Tool",
    "parameters": {
      "query": {
        "type": "string",
        "description": "What to look up",
        "required": true
      }
    }
  }
]
```

The `name` must exactly match the tool name in your OpenClaw workspace — that is the identifier AI Meet sends back over the WebSocket when dispatching a call.

After registering, go to **Persona → OpenClaw tab → Live Tools** to toggle which tools are active and configure per-tool settings.

---

## Workspace Write-Back

After `aimeet_learn_meeting` or `aimeet_learn_all`, write results to these files. These defaults match OpenClaw's standard workspace layout — see **Customization** below if your structure differs.

| Field from learning | Default path |
|---|---|
| `summary` + `decisions` + `action_items` | `memory/YYYY-MM-DD.md` — append under a meeting heading |
| `new_contacts` | `MEMORY.md` under `## Contacts` — check for existing entry by name; update in place rather than duplicate |
| `insights` where `importance == "high"` | `MEMORY.md` under a dated heading |
| `action_items` where `priority == "high"` | `tasks/current.md` if Chief of Staff is installed; otherwise `MEMORY.md` under `## Action Items` |
| `follow_ups` | `memory/YYYY-MM-DD.md` — note inline with the meeting summary |

Running `aimeet_learn_all` twice on the same meetings is safe — results are cached on the AI Meet side.

---

## Customization

### Changing write-back paths

Edit the Workspace Write-Back table above to match your structure. Examples:
- **Obsidian vault:** point `summary` to `Notes/Meetings/YYYY-MM-DD.md`
- **Flat notes folder:** consolidate everything into a single `notes.md`
- **Custom contacts file:** replace `MEMORY.md ## Contacts` with your contacts path

### Adding skill integrations

Declare optional skill dependencies in the frontmatter and add conditional write-back logic. For example, if you use a `todo` skill that writes to `todos/inbox.md`:

```yaml
metadata:
  openclaw:
    optional:
      - chief-of-staff
      - todo
```

Then add to the write-back instructions: *"If the `todo` skill is installed, write high-priority action items to `todos/inbox.md` instead of `MEMORY.md`."*

### Registering additional tools

Add any tool your OpenClaw workspace can execute to the **Tools to Register** section. Common additions: CRM lookups, calendar availability, web research, document search, task creation.

For long-running tools (>10s), set `"is_async": true` and `"estimated_seconds"` — the persona will acknowledge and deliver results when ready rather than waiting inline.

---

## Trigger: User mentions an upcoming meeting

**Phrases:** "I have a call", "jumping on a Zoom", "Teams meeting at", "about to join", "prepping for", "about to meet with", "call with [name]", "standup in [time]", "heading into [meeting name]".

**Action:**
- If the user has a few minutes: call `aimeet_sync_context` with relevant workspace content (SOUL.md, MEMORY.md, relevant contacts, open action items touching the meeting topic).
- Short notice or casual sync: call `aimeet_sync_context_brief` with just attendees and topic.
- Confirm briefly: *"I've briefed [persona name] on your priorities and what I know about [attendees]."*

Do not call `aimeet_sync_context` mid-meeting unless the user explicitly asks.

---

## Trigger: User mentions a meeting just ended

**Phrases:** "just got off a call", "that meeting ended", "we just finished", "I had a call with [name]", "just wrapped", "meeting's done".

**Action:**
- Offer to extract learnings: *"Want me to pull the learnings from that call and update your notes?"*
- On confirmation: call `aimeet_learn_meeting` on the most recent ended meeting (find ID via `aimeet_list_meetings`).
- Write results to workspace files.
- Summarize: new contacts, key decisions, action items assigned to the user.

---

## Trigger: User asks about a past meeting

**Phrases:** "what happened in", "notes from", "pull up [meeting]", "what did we discuss", "transcript from", "[person]'s meeting", "last week's call with", "the [company] call".

**Action:**
1. Call `aimeet_list_meetings` to find the right meeting ID.
2. Call `aimeet_get_meeting` to retrieve full detail (minutes + transcript).
3. Answer the user's specific question. Do not dump the full transcript — summarize what's relevant.

---

## Tool Reference

| Tool | When to call |
|---|---|
| `aimeet_status` | Session start — check for unlearned meetings and last sync time |
| `aimeet_register_tools` | Session start — push tool manifest so the persona knows what it can invoke |
| `aimeet_learn_all` | Session start if `unlearned_meetings > 0` — process all pending meetings |
| `aimeet_learn_meeting` | After a specific meeting ends — extracts contacts, decisions, actions, insights |
| `aimeet_sync_context` | Before a meeting — push full workspace context to the persona |
| `aimeet_sync_context_brief` | Before a short-notice or casual meeting — just attendees + topic |
| `aimeet_list_meetings` | When user asks about past meetings — find the right meeting ID |
| `aimeet_get_meeting` | When user asks about a specific meeting — get full transcript + minutes |
| `aimeet_upcoming` | Session start — check for calendar-connected meetings starting soon |
| `aimeet_integration_guide` | When user wants to connect a skill to live meetings — returns setup instructions and examples. Pass `skill_name` for a specific skill (e.g. `"github"`), or omit for the full guide. |

---

## Trigger: User wants to connect a skill to live meetings

**Phrases:** "add [skill] to my persona", "can my persona use [tool]", "wire up [skill] to AI Meet", "I want [persona] to be able to [capability] during calls", "how do I integrate [skill]", "connect [skill] to meetings".

**Action:**
1. Call `aimeet_integration_guide` — pass `skill_name` if the user named a specific skill, otherwise omit it.
2. If the guide has a pre-built example for that skill: walk the user through copying the JSON into `SKILL.md` under **Tools to Register** and call `aimeet_register_tools`.
3. If no example exists (undocumented or custom skill): use the "ask your agent" pattern from the guide — inspect the skill's `SKILL.md`, identify invocable functions, write tool definitions, add them to the aimeet `SKILL.md`, then call `aimeet_register_tools`.
4. Confirm: *"[Tool names] are now registered. Go to Persona → OpenClaw tab → Live Tools to toggle them on."*

Do not ask the user to look up tool names manually — read the skill files yourself and propose the definitions.

---

## What NOT to Do

- Do not announce tool calls. Don't say "calling aimeet_status now…". Just call it.
- Do not fabricate meeting content. If no meetings exist or the transcript isn't ready, say so.
- Do not call `aimeet_sync_context` mid-meeting unless the user explicitly asks.
- Do not call `aimeet_learn_all` more than once per session start.
- Do not write duplicate entries to workspace files — check for existing content first.
