---
name: multi-workplace
description: >
  Manage multiple workplaces (project directories) with multi-agent orchestration,
  isolated memory, and inter-agent communication. Use when the user mentions:
  workplace init/list/switch/scan/status/agents/export/import, managing projects,
  switching between codebases, multi-agent workflows, agent handoff, kernel agent,
  workspace structure, deploy environments, or any variation of "workplace" commands.
  Auto-detects .git folders as workplaces. Each workplace has its own agents, memory,
  skills, and deployment configs in a .workplace/ directory. Syncs context to
  Cursor (.cursor/rules), Claude Code (CLAUDE.md), and OpenCode (opencode.jsonc).
---

# Multi-Workplace Skill

Manage multiple project workplaces with per-workspace agents, isolated memory, and Swarm-style agent orchestration.

## Quick Reference

| Command | Action |
|---------|--------|
| `workplace init [path]` | Initialize workplace (scan existing or set up new) |
| `workplace list` | List all workplaces (inline buttons to switch) |
| `workplace switch <name\|uuid>` | Switch active workplace |
| `workplace scan [path]` | Discover .git workplaces in subdirectories |
| `workplace link <path>` | Link a related workplace |
| `workplace unlink <path\|uuid>` | Remove a linked workplace |
| `workplace status` | Current workplace info + agent status |
| `workplace agents` | List agents in current workplace |
| `workplace agent start <name>` | Start an agent (runs as sub-agent) |
| `workplace agent stop <name>` | Stop a running agent |
| `workplace kernel start` | Start persistent kernel agent |
| `workplace kernel stop` | Stop kernel agent |
| `workplace export [zip\|json]` | Export workplace config |
| `workplace import <file>` | Import workplace from export |
| `workplace delete <name\|uuid>` | Remove from registry |
| `workplace deploy <env>` | Show/run deploy instructions |
| `workplace sync <ide>` | Generate context for cursor/claude/opencode/all |

## Architecture

### Registry

Central registry at `~/.openclaw/workspace/.workplaces/`:
- `registry.json` — all known workplaces with UUID, path, hostname, links
- `current.json` — currently active workplace

### Per-Workplace Structure

Each project gets a `.workplace/` directory:

```
.workplace/
├── config.json          # UUID, name, path, hostname, linked, parent
├── agents/*.md          # Agent role definitions (kernel.md always present)
├── memory/              # Isolated daily logs (YYYY-MM-DD.md)
├── skills/              # Workplace-specific skills (user-managed via git)
├── chat.md              # Inter-agent communication
├── structure.json       # Auto-scanned file tree
├── full-tree.md         # Full tree with parent + linked workplaces (by hostname)
├── process-status.json  # Agent runtime states and errors
└── deploy/              # Deployment docs: dev.md, main.md, pre.md
```

### Workplace Detection

- Any directory with `.git/` is a potential workplace
- Submodules included as nested workplaces
- Parent workplace auto-detected from parent directories
- Manual linking via `workplace link`

## Workflows

### Initialize a Workplace

1. Run `scripts/init_workplace.sh <path> [--name <name>] [--desc <desc>]`
2. For existing projects: scan file structure, read `*.md` files, analyze project type, suggest agents
3. For empty folders: ask project name, description, language/framework, roles needed
4. Creates `.workplace/` structure, registers in central registry, sets as current
5. See [init-guide.md](references/init-guide.md) for full flow details

### Agent System

Agents are defined as `.md` files in `.workplace/agents/` with YAML frontmatter (name, role, triggers, handoff_to). Run agents via `sessions_spawn` with system prompts built from their definitions + workplace context.

- See [agent-system.md](references/agent-system.md) for agent creation, Swarm handoff, and runtime details

### Inter-Agent Communication

Agents communicate via `chat.md` using a structured message protocol. The Rust file-watcher server monitors changes and outputs parsed messages as JSON lines.

- See [chat-protocol.md](references/chat-protocol.md) for message format spec

### Rust File-Watcher Server

Binary at `assets/bin/workplace-server-{os}-{arch}`. Build from source with `scripts/build.sh`.

```bash
# Start server for a workplace
workplace-server /path/to/project

# Server outputs JSON lines to stdout for each new chat.md message
{"timestamp":"...","sender":"coder","recipient":"reviewer","broadcast":[],"message":"...","line_number":1}
```

### Export/Import

- **ZIP**: Full `.workplace/` folder (memory excluded by default)
- **JSON**: Config + agent definitions + deploy docs as portable manifest
- Import generates a new UUID to avoid collisions

## IDE Integration

Sync workplace context to external coding tools:

- **Cursor** → `.cursor/rules/workplace.mdc` (MDC with frontmatter)
- **Claude Code** → `CLAUDE.md` (markdown, marker-based updates)
- **OpenCode** → `opencode.jsonc` instructions field

Run `workplace sync all` to update all detected IDEs, or target one: `workplace sync cursor`.

See [ide-sync.md](references/ide-sync.md) for implementation details.

## Scripts

| Script | Purpose |
|--------|---------|
| `scripts/init_workplace.sh` | Initialize .workplace/ in a directory |
| `scripts/scan_workplaces.sh` | Find .git workplaces under a path |
| `scripts/build.sh` | Build Rust server for current platform |

## Supermemory Integration

Each workplace uses its UUID as `containerTag` for supermemory operations:
- Kernel agent saves structure summaries and project facts
- All workplace memories are isolated by containerTag
- Enables cross-session project state awareness

## Command Details

See [commands.md](references/commands.md) for full command reference with examples.
