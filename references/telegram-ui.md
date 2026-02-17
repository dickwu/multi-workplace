# Telegram UI Integration

## Workspace Switcher

When the user runs `workplace list` or `workplace switch` from Telegram, present an interactive message with inline buttons.

### Showing the List

Read `~/.openclaw/workspace/.workplaces/registry.json` and `current.json`, then send a message with inline buttons:

```
📁 **Workplaces** — current: **{current_name}**

Switch to:
```

Buttons: one per registered workplace.
- Current workplace: `disabled: true`, `style: "secondary"`
- Other workplaces: `style: "primary"`
- Button label: emoji + workspace name
- Button callback triggers: `workplace switch {uuid}`

### Implementation

Use the `message` tool with `components`:

```json
{
  "action": "send",
  "message": "📁 **Workplaces** — current: **logstream-dashboard**\n\nSwitch to:",
  "components": {
    "blocks": [{
      "type": "buttons",
      "buttons": [
        {"label": "⚙️ logstream", "style": "primary"},
        {"label": "🌐 logstream-dashboard", "style": "secondary", "disabled": true}
      ]
    }]
  }
}
```

When a button is clicked, the callback text arrives as a normal user message. Handle it by:

1. Matching the workspace name from the button label
2. Updating `current.json` with the selected workspace UUID and path
3. Updating `lastActive` in `registry.json`
4. Sending a confirmation message with the new workspace context

### Switch Confirmation

After switching, send:

```
✅ Switched to **{name}**
📂 `{path}`
🔗 Linked: {linked workplaces or "none"}

Agents: {comma-separated agent names}
```

### Workspace Status Card

For `workplace status`, send a formatted card:

```
📁 **{name}** ({uuid_short}...)
📂 `{path}`
🖥️ Host: {hostname}
🔗 Linked: {linked list}

**Agents:**
{for each agent: emoji + name — role (status)}

**Deploy:** dev | main | pre
```

### Agent Control Buttons

For `workplace agents`, show agents with start/stop buttons:

```json
{
  "blocks": [
    {"type": "text", "text": "**Agents for logstream:**"},
    {"type": "buttons", "buttons": [
      {"label": "▶️ Start rust-dev", "style": "success"},
      {"label": "▶️ Start reviewer", "style": "success"},
      {"label": "▶️ Start sdk-dev", "style": "success"},
      {"label": "🔄 Start kernel", "style": "primary"}
    ]}
  ]
}
```

### Deploy Environment Buttons

For `workplace deploy`, show environment options:

```json
{
  "blocks": [{
    "type": "buttons",
    "buttons": [
      {"label": "🛠️ dev", "style": "secondary"},
      {"label": "🚀 main", "style": "danger"},
      {"label": "🧪 pre", "style": "primary"}
    ]
  }]
}
```

## Callback Handling

When a button click arrives as a user message, detect the pattern and route:

| Button text pattern | Action |
|---|---|
| Contains workspace name from registry | `workplace switch {matched_uuid}` |
| "Start {agent}" | `workplace agent start {agent}` |
| "Stop {agent}" | `workplace agent stop {agent}` |
| "dev" / "main" / "pre" | `workplace deploy {env}` |

## Platform Detection

Only use inline buttons on platforms that support them:
- **Telegram** ✅ inline buttons via `components`
- **Discord** ✅ buttons via components
- **WhatsApp** ❌ use numbered list instead
- **Signal** ❌ use numbered list instead

Check the `channel` from inbound metadata. If the channel doesn't support buttons, fall back to a numbered text list where the user replies with a number.
