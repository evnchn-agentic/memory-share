# memory-share

A Claude Code skill for synchronizing AI agent memory across a multi-node homelab via SSH.

## What it does

Claude Code's memory system (`~/.claude/projects/<path>/memory/`) is per-machine. If you run agents on multiple nodes, each agent starts with only its own history. This skill fixes that.

`/memory-sync` lets any Claude Code instance pull memory from all nodes, merge intelligently (AI reads content, not just timestamps), and push the unified knowledge back — so every agent across the fleet benefits from every other agent's discoveries.

## Why not cron?

I tried a cron-based rsync approach on a central hub node. It:
- Was a single point of failure
- Used dumb mtime-based merging (silent overwrites)
- Failed silently when nodes were unreachable
- Required a dedicated always-on node

The skill approach is better because:
- **Any node can run it** — no single point of failure
- **AI-powered merge** — Claude reads diverged files and merges content intelligently
- **On-demand** — run when you need it, not on a blind schedule
- **Self-documenting** — the skill explains itself to the agent running it

## Installation

```bash
mkdir -p ~/.claude/skills/memory-sync
cp SKILL.md ~/.claude/skills/memory-sync/SKILL.md
```

Then create your node registry:

```bash
cat > ~/.claude/skills/memory-sync/nodes.conf << 'EOF'
# name          ip              user        memory_path
my-server       192.168.1.10    alice       ~/.claude/projects/-home-alice/memory/
my-macbook      192.168.1.20    alice       ~/.claude/projects/-Users-alice/memory/
EOF
```

Then in any Claude Code session: `/memory-sync --status`

## Usage

| Command | Description |
|---|---|
| `/memory-sync` | Full sync: pull from all nodes, merge, push back |
| `/memory-sync --status` | Show file counts and sync state per node |
| `/memory-sync --pull-only` | Pull and compare without pushing |
| `/memory-sync --dry-run` | Show what would change without doing it |
| `/memory-sync .68` | Only sync with a specific node |
| `/memory-sync .68 .34` | Sync with specific nodes |

## Prerequisites

- Claude Code installed on the invoking machine
- SSH access to target nodes (credentials in `~/.claude/CLAUDE.md`)
- `sshpass` and `rsync` available
- Memory directories at `~/.claude/projects/-home-<user>/memory/` (Linux) or `~/.claude/projects/-Users-<user>/memory/` (macOS)

## Safety

- **Never touches conversation history** — only the `memory/` subdirectory
- **Additive by default** — no blanket rsync `--delete`; the only removals are user-confirmed deletions (surfaced, queried, tombstoned), never inferred from an unreachable node
- **Backs up first-time nodes** before their first sync
- **AI reads diverged files** before deciding — no silent overwrites

## Customization

The node registry lives in `nodes.conf` alongside `SKILL.md` — edit it to match your infrastructure. Credentials are intentionally NOT stored in the skill — they come from `CLAUDE.md` which is loaded into every Claude Code session.

## Part of the evnchn-agentic fleet

This skill is one piece of a multi-node homelab where AI agents operate autonomously on dedicated hardware. See [evnchn-agentic](https://github.com/evnchn-agentic) for the full picture.
