# Telegram UI Integration

## /workplace Command — Hierarchical Navigation

### Top Level (no args or `/workplace`)

Show **parent workspaces and standalone workplaces** as the first level. Group by: top-level items (no parent) first.

Read `registry.json`, separate into:
- **Parents / standalone**: entries where `parent == null`
- **Children**: entries where `parent != null` (shown when user drills into a parent)

**Top-level message:**

```
📁 **Workplaces**
Current: **{current_name}** {current_path_short}
```

**Buttons:** One row per top-level workspace. Parent workplaces show children count.

```json
{
  "blocks": [{
    "type": "buttons",
    "buttons": [
      {"label": "📂 log-stream (2)", "style": "primary"},
      {"label": "🔧 multi-workplace ✓", "style": "secondary", "disabled": true}
    ]
  }]
}
```

- Current workspace (or its parent): `disabled: true`, `style: "secondary"`, append ` ✓`
- Parent workplaces: show `(N)` child count
- Standalone workplaces: no count

### Drill into Parent

When user clicks a parent workspace button (e.g. "📂 log-stream (2)"), show its children:

```
📂 **log-stream** — parent workspace
`/Users/.../opensource/log-stream`
```

**Buttons:** One per child + a "Use parent" option + back button.

```json
{
  "blocks": [{
    "type": "buttons",
    "buttons": [
      {"label": "⚙️ logstream", "style": "primary"},
      {"label": "🌐 logstream-dashboard ✓", "style": "secondary", "disabled": true},
      {"label": "📂 Use log-stream (parent)", "style": "secondary"},
      {"label": "← Back", "style": "secondary"}
    ]
  }]
}
```

- Current child: `disabled: true` with ` ✓`
- "Use parent" button: switches context to the parent workspace itself
- "← Back" button: returns to top-level list

### Colon Syntax — Direct Navigation

Support `parent:child` syntax for quick switching without menus:

```
/workplace log-stream:logstream          → switch to logstream under log-stream
/workplace log-stream:logstream-dashboard → switch to logstream-dashboard
/workplace log-stream                     → show log-stream's children (drill-in)
/workplace multi-workplace                → switch directly (standalone, no children)
```

**Resolution logic:**
1. If input contains `:`, split into `parent:child`
2. Find parent by name in registry (fuzzy match OK)
3. Find child by name where `child.parent == parent.uuid`
4. Switch to child

If no `:`, check if the name matches a parent with children → show drill-in view.
If it matches a standalone or child → switch directly.

### Switch Confirmation

After switching, send:

```
✅ Switched to **logstream**
📂 `/Users/.../log-stream/logstream`
📂 Parent: log-stream
🔗 Linked: logstream-dashboard

Agents: kernel, rust-dev, sdk-dev, reviewer, publisher
```

### Button Callback Routing

| Button text | Action |
|---|---|
| `📂 {parent} (N)` | Show parent's children (drill-in) |
| `⚙️/🌐/🔧 {name}` | Switch to that workspace |
| `📂 Use {name} (parent)` | Switch to parent workspace |
| `← Back` | Show top-level list |
| `📂 {name}` (loaded view) | Switch to loaded workplace |
| `➕ Load workplace` | Prompt for path/name |
| `➖ Unload workplace` | Show unload picker |
| `❌ {name}` | Unload that workplace |
| `▶️ Start {agent}` | `workplace agent start {agent}` |
| `⏹ Stop {agent}` | `workplace agent stop {agent}` |

### Agent and Deploy Buttons

Same as before — shown after switching or via `/workplace agents` / `/workplace deploy`:

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

### Loaded Workplaces

For `/workplace loaded`:

```
📂 **Loaded Workplaces** (2)
Active: **multi-workplace**
```

**Buttons:** One row per loaded workplace + management buttons.

```json
{
  "blocks": [
    {"type": "text", "text": "📂 **Loaded Workplaces** (2)\nActive: **multi-workplace**"},
    {"type": "buttons", "buttons": [
      {"label": "📂 log-stream", "style": "primary"},
      {"label": "🔧 multi-workplace ✓", "style": "secondary", "disabled": true}
    ]},
    {"type": "buttons", "buttons": [
      {"label": "➕ Load workplace", "style": "success"},
      {"label": "➖ Unload workplace", "style": "danger"}
    ]}
  ]
}
```

- Current workplace: `disabled: true` with ` ✓`
- Clicking a loaded workplace switches to it
- "➕ Load workplace" prompts for path/name/uuid
- "➖ Unload workplace" shows loaded list with unload buttons

### Load Confirmation

After loading a workplace:

```
✅ Loaded: **log-stream**
📂 `/Users/.../opensource/log-stream`
🔗 Also linked to current workplace

Loaded workplaces: 2
```

### Unload Flow

When user clicks "➖ Unload workplace", show loaded workplaces with unload buttons:

```json
{
  "blocks": [
    {"type": "text", "text": "Select workspace to unload:"},
    {"type": "buttons", "buttons": [
      {"label": "❌ log-stream", "style": "danger"},
      {"label": "← Back", "style": "secondary"}
    ]}
  ]
}
```

### Button Callback Routing (Loaded)

| Button text | Action |
|---|---|
| `📂 {name}` (in loaded view) | Switch to that loaded workplace |
| `➕ Load workplace` | Prompt for path/name |
| `➖ Unload workplace` | Show unload picker |
| `❌ {name}` | Unload that workplace |

### Status Card

For `/workplace status`:

```
📁 **logstream** (93cb20c8...)
📂 `/Users/.../log-stream/logstream`
🖥️ Host: dsgnmac2
📂 Parent: log-stream (74cdd6fd...)
🔗 Linked: logstream-dashboard

**Agents:**
🟢 kernel — persistent structure watcher
⚪ rust-dev — Rust systems developer
⚪ reviewer — code reviewer
⚪ publisher — release manager

**Loaded:** log-stream, multi-workplace
**Deploy:** dev | main | pre
```

## Platform Fallback

On platforms without inline buttons (WhatsApp, Signal), show hierarchical text:

```
📁 Workplaces (current: logstream)

1. 📂 log-stream (parent)
   1a. ⚙️ logstream ← current
   1b. 🌐 logstream-dashboard
2. 🔧 multi-workplace

Reply with number (e.g. "1b") or name (e.g. "log-stream:logstream-dashboard")
```

For `/workplace loaded` on platforms without buttons:

```
📂 Loaded Workplaces (2)
Active: multi-workplace

1. log-stream — /Users/.../opensource/log-stream
2. multi-workplace — /Users/.../workspace/multi-workplace ← current

Commands: "workplace load <path>" / "workplace unload <name>"
```
