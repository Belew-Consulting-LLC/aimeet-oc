---
name: aimeet-live
description: "Connects your workspace to AI Meet Live. Check for unlearned meetings silently at every session start and extract them automatically. When the user mentions an upcoming meeting, sync workspace context to their AI persona. When they mention a call just ended, offer to extract learnings. When asked about a past meeting, retrieve it directly. All MCP tools are available via the aimeet MCP server — do not announce tool calls, just do them."
version: 1.0.0
emoji: "🎙️"
homepage: https://aimeet.live/docs/openclaw
metadata:
  openclaw:
    requires:
      config:
        - mcp.servers.aimeet
---

## Access Control (configure in persona settings)

Before live tool calling works during meetings, configure two settings in the AI Meet Live dashboard under **Persona → OpenClaw tab → Live Tools**:

**Speaker gate** — who can trigger tool calls:
- `authorized_only` (default) — only speakers on your persona's Authorized Speakers list can invoke tools
- `everyone` — any meeting participant's utterance can trigger tools

**Trigger phrase** — optional keyword that must appear in the utterance before tools are injected. Example: if you set the trigger phrase to `"hey beau"`, the participant must say something like *"Hey Beau, look up my notes on Acme"* — utterances without the phrase won't invoke tools. Leave blank to allow any utterance to trigger tools (subject to the speaker gate).

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

---

## Tools to Register

Edit this list to include the tools from your OpenClaw workspace you want the AI Meet persona to invoke during live meetings. The agent reads these definitions at every session start and calls `aimeet_register_tools` silently. Registration is idempotent — safe to run every session.

```json
[
  {
    "name": "example_tool",
    "description": "Replace this with your actual tool — what it does, when to use it. The LLM reads this description to decide whether to call the tool.",
    "display_name": "Example Tool",
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

After registration, go to **Persona → OpenClaw tab → Live Tools** in the AI Meet dashboard to toggle which tools are active for each persona.

---

## Session Start Behavior (do this silently every time)

At the start of every session, without announcing it:

1. Call `aimeet_status`
2. If `unlearned_meetings > 0`: call `aimeet_learn_all` immediately and write the results into workspace files (see write-back map below). Do not narrate this process — just do it.
3. Call `aimeet_register_tools` with the tool definitions from the **Tools to Register** section above.
4. If `aimeet_upcoming` returns meetings within the next 60 minutes: surface them naturally once — e.g. *"You have a call with Sarah in 40 minutes — want me to brief [persona name]?"*

Do not call `aimeet_learn_all` more than once per session start — results are cached.

---

## Trigger: User mentions an upcoming meeting

**Phrases that should trigger this:** "I have a call", "jumping on a Zoom", "Teams meeting at", "about to join", "prepping for", "about to meet with", "call with [name]", "standup in [time]", "heading into [meeting name]".

**Action:**
- If the user has a few minutes: call `aimeet_sync_context` with relevant workspace content (SOUL.md, MEMORY.md, relevant contacts from known_people.md, any open action items that touch the meeting topic).
- If it's short-notice or a casual internal sync: call `aimeet_sync_context_brief` with just attendees and topic.
- Confirm briefly: *"I've briefed [persona name] on your priorities and what I know about [attendees]."*

Do not call `aimeet_sync_context` mid-meeting unless the user explicitly asks.

---

## Trigger: User mentions a meeting just ended

**Phrases that should trigger this:** "just got off a call", "that meeting ended", "we just finished", "I had a call with [name]", "just wrapped", "meeting's done".

**Action:**
- Offer to extract learnings: *"Want me to pull the learnings from that call and update your notes?"*
- On confirmation: call `aimeet_learn_meeting` on the most recent ended meeting (find the ID via `aimeet_list_meetings`).
- Write results into workspace files (see write-back map below).
- Summarize what was extracted: new contacts, key decisions, action items assigned to the user.

---

## Trigger: User asks about a past meeting

**Phrases that should trigger this:** "what happened in", "notes from", "pull up [meeting]", "what did we discuss", "transcript from", "[person]'s meeting", "last week's call with", "the [company] call".

**Action:**
1. Call `aimeet_list_meetings` to find the right meeting ID.
2. Call `aimeet_get_meeting` to retrieve the full detail (minutes + transcript).
3. Answer the user's specific question from the content. Do not dump the full transcript — summarize what's relevant.

---

## Workspace Write-Back Map

After `aimeet_learn_meeting` or `aimeet_learn_all`, write results into these files:

| Field from learning | Write to |
|---|---|
| `summary` + `decisions` + `action_items` | `memory/YYYY-MM-DD.md` (one file per meeting date) |
| `new_contacts` | `life/known_people.md` — check for existing entry by name before appending; update in place rather than duplicate |
| `insights` where `importance == "high"` | `MEMORY.md` under a dated heading |
| `action_items` where `priority == "high"` | `life/active_projects.md` |
| `follow_ups` | `life/follow_ups.md` — include `with_whom` and `timing` |

Running `aimeet_learn_all` twice on the same meetings is safe — results are cached on the AI Meet side.

---

## Tool Reference

| Tool | When to call |
|---|---|
| `aimeet_status` | Session start — check for unlearned meetings and last sync time |
| `aimeet_register_tools` | Session start — push your tool manifest so the persona knows what it can invoke |
| `aimeet_learn_all` | Session start if `unlearned_meetings > 0` — process all pending meetings at once |
| `aimeet_learn_meeting` | After a specific meeting ends — extracts contacts, decisions, actions, insights |
| `aimeet_sync_context` | Before a meeting — push full workspace context to the persona |
| `aimeet_sync_context_brief` | Before a short-notice or casual meeting — just attendees + topic |
| `aimeet_list_meetings` | When user asks about past meetings — find the right meeting ID |
| `aimeet_get_meeting` | When user asks about a specific meeting — get full transcript + minutes |
| `aimeet_upcoming` | Session start — check for calendar-connected meetings starting soon |

---

## What NOT to Do

- Do not announce each tool call. Don't say "calling aimeet_status now…". Just call it.
- Do not fabricate meeting content. If no meetings exist or the transcript isn't ready yet, say so.
- Do not call `aimeet_sync_context` mid-meeting unless the user explicitly asks.
- Do not call `aimeet_learn_all` more than once per session start.
- Do not write duplicate entries into workspace files — check for existing content first.
