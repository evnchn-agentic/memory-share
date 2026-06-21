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
5. Categorize each unique filename by how many reachable nodes have it:
   - **Identical**: `md5`/`shasum` matches across all sources -> skip
   - **Diverged**: present on multiple sources, contents differ -> intelligent merge (Phase 3)
   - **Partial**: present on some reachable nodes but absent from others -> AMBIGUOUS (newly created on one node, OR deleted on the others). Resolve in **Phase 2.5** — never blindly redistribute (silent resurrection) and never blindly delete.

If `--pull-only` was requested, report the inventory and stop.
If `--dry-run` was requested, show what would be pushed and stop.

### Phase 2.5: Deletion detection (surface & confirm)

A **Partial** file (present on some reachable nodes, missing from others) is the one case a naive additive sync gets wrong: it redistributes the file back to the nodes it was deleted from, silently **resurrecting** it. Handle it explicitly:

1. **List every Partial file** with where it's present vs missing, e.g. `foo.md: on [a, b, c], missing from [d]`.
2. **Ask the user, per file (or in one batch):** was this *newly created* (→ distribute to all) or *deleted* (→ remove everywhere)? When unsure, default to **distribute** — additive is the safe default; never delete on a guess.
3. **Only reachable nodes count.** A file missing from an *unreachable* node is NOT a deletion signal — you cannot see that node, so you cannot know. Never delete based on an unreachable node.
4. **For confirmed deletions:** `rm -f <memory_path>/<file>` on every reachable node, and **record a tombstone** so it cannot resurrect from a node that was offline during the delete.

**Tombstone ledger:** keep a synced `MEMORY.deleted` file (one `filename<TAB>iso8601` per line) in the memory dir alongside the `.md` files. On every sync, before distributing Partial/New files, skip any filename listed there — UNLESS a copy now has an mtime newer than the tombstone (a genuine recreate), in which case drop the tombstone and treat it as a normal new file. This makes deletions durable across nodes that rejoin later, while still allowing intentional re-creation.


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
2. For each reachable node, rsync canonical to its memory path additively (no rsync `--delete`). The ONLY removals are the deletions the user confirmed in Phase 2.5 — applied as targeted `rm -f` of those specific files (plus tombstone enforcement), never a blanket `--delete`.
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
2. **NEVER blanket-`--delete` on rsync push** — additive by default. The only removals are deletions the user explicitly confirmed in Phase 2.5 (surfaced, queried, tombstoned); never infer a deletion from an unreachable node
3. **NEVER overwrite without reading** — if a file diverges, read both versions before deciding
4. **Back up first-time nodes** — if a node has no MEMORY.md in its memory path, back up `~/.claude/` before first push
5. **Path encoding** — Linux uses `-home-<user>`, macOS uses `-Users-<user>`. Use what's in nodes.conf. Never mix them up.
6. **Report everything** — no silent operations. The user should see exactly what changed.
