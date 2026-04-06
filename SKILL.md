---
name: memory-sync
description: Synchronize memory files across all homelab nodes via SSH. Pulls, merges intelligently, pushes back.
disable-model-invocation: false
user-invocable: true
argument-hint: "[.68|.34|all] [--status|--pull-only|--dry-run]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent
---

# Memory Sync

Synchronize Claude Code memory directories across all nodes in your fleet.

Memory paths follow Claude Code's convention:
- Linux: `~/.claude/projects/-home-<user>/memory/`
- macOS: `~/.claude/projects/-Users-<user>/memory/`

## Current node status

!`cat ${CLAUDE_SKILL_DIR}/nodes.conf 2>/dev/null | grep -v '^#' | grep -v '^$' | while read name ip user path; do (ping -c1 -W1 $ip &>/dev/null && echo "${name} (${ip}) UP" || echo "${name} (${ip}) DOWN") & done; wait 2>/dev/null`

## User request: $ARGUMENTS

## Node registry

Loaded from `nodes.conf` in the skill directory (`${CLAUDE_SKILL_DIR}/nodes.conf`).

If `nodes.conf` does not exist, prompt the user to create one. Format:

```
# name          ip              user        memory_path
# Lines starting with # are comments
my-server       192.168.1.10    alice       ~/.claude/projects/-home-alice/memory/
my-macbook      192.168.1.20    alice       ~/.claude/projects/-Users-alice/memory/
```

Read the file to discover which nodes to sync:
!`cat ${CLAUDE_SKILL_DIR}/nodes.conf 2>/dev/null || echo "NO_NODES_CONF"`

## SSH credentials

Credentials are NOT stored in this skill. Obtain them from CLAUDE.md (loaded into every session) which documents SSH patterns and passwords for your nodes.

If your CLAUDE.md doesn't document SSH access, ask the user how to connect to their nodes.

## Sync procedure

### Phase 1: Discovery
1. Check which nodes are UP (pre-populated above via dynamic context)
2. If user specified specific nodes, filter to those only
3. For each reachable node, SSH in and list memory files with sizes and mtimes (use the memory_path from nodes.conf):
   - Linux: `stat -c '%n %s %Y' <memory_path>/*.md 2>/dev/null`
   - macOS: `stat -f '%N %z %m' <memory_path>/*.md 2>/dev/null`
4. Also list the local (invoking machine's) memory files for comparison
5. Report summary table: node, file count, total size, last modified

If `--status` was requested, stop here and report.

### Phase 2: Pull & Compare
1. Stage locally: `/tmp/memory-sync-staging/<node>/`
2. For each reachable node, rsync its memory dir into staging (use SSH pattern from CLAUDE.md):
   ```
   rsync -av --exclude='*.jsonl' --exclude='.history' <node-memory-path>/ /tmp/memory-sync-staging/<node>/
   ```
3. Also copy the local memory dir into staging as `local/`
4. Build inventory: for each unique `.md` filename, list which sources have it
5. Categorize:
   - **New**: only exists on one source -> distribute to all
   - **Identical**: `md5`/`shasum` matches across all sources -> skip
   - **Diverged**: different content -> needs intelligent merge

If `--pull-only` was requested, report the inventory and stop.
If `--dry-run` was requested, show what would be pushed and stop.

### Phase 3: Intelligent Merge
For **new** files: accept as-is into canonical (`/tmp/memory-sync-staging/canonical/`).

For **diverged** files: **READ both versions**. You are an AI — don't just pick by mtime. Instead:
- If one version is a superset (has all content of the other plus more), use the superset
- If both have unique additions, merge the content intelligently
- If they genuinely conflict (contradictory facts), present both versions to the user and ask
- Write merged result to `/tmp/memory-sync-staging/canonical/`
- NEVER silently discard content

For MEMORY.md specifically: merge index entries from all sources, deduplicate, maintain alphabetical ordering within sections.

### Phase 4: Push
1. **First-time nodes** (no MEMORY.md in their memory path): back up their entire `~/.claude/` to `/tmp/memory-sync-backup-<node>/` first
2. For each reachable node, rsync canonical to its memory path. Use `--no-delete` flag — additive only, NEVER remove files from target nodes.
3. Spot-check: read one file from each node to verify push succeeded

### Phase 5: CLAUDE.md sync
1. Source of truth: the invoking machine's `~/.claude/CLAUDE.md`
2. SCP it to every reachable node's `~/.claude/CLAUDE.md`

### Phase 6: Report
Print a summary:
- Nodes synced (and any that were unreachable)
- Files added to each node
- Files updated (with brief description of what changed)
- Any conflicts that needed user input
- Any errors encountered

## CRITICAL SAFETY RULES

1. **NEVER touch conversation history** — no `.jsonl` files, no `.history/` dirs, no `projects/*/` anything except `memory/`
2. **NEVER use `--delete` on rsync push** — additive only, always
3. **NEVER overwrite without reading** — if a file diverges, read both versions before deciding
4. **Back up first-time nodes** — if a node has no MEMORY.md in its memory path, back up `~/.claude/` before first push
5. **Path encoding** — Linux uses `-home-<user>`, macOS uses `-Users-<user>`. Use what's in nodes.conf. Never mix them up.
6. **Report everything** — no silent operations. The user should see exactly what changed.
