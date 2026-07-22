---
name: codex-sub-agent
description: Manual-only supervisor that delegates difficult reasoning and routine execution to an equivalent pool of external AI CLIs, preserves continuing conversations, allows at most two safe parallel workers, handles temporary quota outages, and keeps all Git mutations under Codex control. Use only when the user explicitly invokes $codex-sub-agent.
---

# Codex Sub Agent

Delegate substantive work to external CLIs. Codex routes, manages Git, verifies results, and reports.

## CLI pool configuration

This is the only place that defines pool membership. Add or remove a CLI here; all entries are otherwise treated equally.

Run commands from the target workspace; set the shell working directory or `cd` first. AGY has no `--cwd` flag. Pool commands use headless print mode and skip interactive permission prompts.

| CLI | Deep command | Fast command |
|---|---|---|
| `agy` | `agy --dangerously-skip-permissions {SESSION} --model claude-opus-4-6-thinking -p "{PROMPT}"` | `agy --dangerously-skip-permissions {SESSION} --model gemini-3.6-flash-high -p "{PROMPT}"` |
| `agy2` | `agy2 --dangerously-skip-permissions {SESSION} --model claude-opus-4-6-thinking -p "{PROMPT}"` | `agy2 --dangerously-skip-permissions {SESSION} --model gemini-3.6-flash-high -p "{PROMPT}"` |

Use safe shell quoting. Each added CLI must provide fixed deep and fast commands.

Set `{SESSION}` to:

- empty for a new conversation;
- `--conversation "{CONVERSATION_ID}"` to continue an exact conversation;
- `--continue` only when the latest conversation for that pool member and workspace is certainly the intended workstream.

## Routing and conversations

- Codex handles lightweight management, all Git mutations, targeted verification, and requested commits.
- Deep model: novel ideas, hard reasoning, architecture, experiment design, ambiguous decisions, difficult diagnosis, and surprising results.
- Fast model: implementation, search, debugging, tests, experiments, data processing, and routine edits.
- Mixed task: normally obtain one concise deep plan, then implement with the fast model. Reuse the same conversation when context helps; changing the model does not require a new workstream.
- Give each coherent task a stable workstream identity and a meaningful delegation unit.
- Prefer exact continuation by conversation ID. Codex should retain a local mapping of workspace, pool member, workstream, and conversation ID when the ID is available from CLI metadata or cache.
- Never use `--continue` after another workstream has used the same pool member and workspace, or when parallel calls make the latest conversation ambiguous.
- If the intended conversation ID is unknown and `--continue` is ambiguous, start a new conversation with a concise handoff rather than risk replying in the wrong thread.
- After switching pool members, start a new conversation with a concise handoff; conversations are not shared across members.

## Git boundary

- External agents may inspect Git read-only, such as `status`, `diff`, `log`, `show`, branch/tag lists, and `worktree list`.
- Only Codex may change Git state: index, commits, branches, refs, tags, stashes, remotes, submodules, worktrees, configuration, hooks, or `.git` metadata.
- Agents must not use mutating commands such as `add`, `commit`, `checkout`, `switch`, `restore`, `reset`, `revert`, `clean`, `merge`, `rebase`, `stash`, `pull`, `fetch`, `push`, branch/tag creation or deletion, worktree mutation, or submodule updates. If a command is not clearly read-only, do not run it.
- Codex records the baseline before delegation and performs the authoritative status and diff review afterward.

Every workspace delegation must include:

```text
Read-only Git inspection is allowed. Do not run Git commands that change repository state; Codex manages all Git mutations.
```

## Parallelism

- Maximum: two external CLI processes. Default: one.
- Use two only for independent tasks that save meaningful time and fit available memory; serialize memory-heavy work or when the machine is under pressure.
- Never parallelize dependent stages, overlapping file writes, shared mutable outputs, conflicting experiments, or shared Git mutation.
- For two writers, Codex first provides separate worktrees or clearly disjoint directories. Only Codex may create or manage worktrees.
- Run at most one process per pool member. Give parallel agents distinct workstreams and non-overlapping scopes; Codex integrates after both finish.

## Delegation prompt

Send only the objective, constraints, relevant paths, assigned scope, completion criteria, concise handoff when needed, and the Git boundary. Let the agent inspect the workspace; do not paste large files, logs, diffs, or the whole chat.

Require autonomous work, checks, and a final block of at most 12 lines:

```text
STATUS: done | blocked | failed
SUMMARY: brief result
CHANGED: files or none
CHECKS: commands and outcomes; Git commands must be read-only
NEXT: none or one required next action
```

Deep planning output must be at most 250 words.

## Availability, switching, and takeover

- Start a new independent workstream on any available pool member; conversation continuity decides where continuing work stays.
- If the current member hits a quota, rate limit, unavailable model, authentication error, or executable failure, switch to another available member with a concise handoff.
- Codex takes over only when no pool member is available. Continue from the current workspace and process state; do not restart completed work blindly.
- When every member is quota-limited, the task is non-urgent, no delegated responsibility remains active, and quota is expected to refresh within two hours, Codex may wait and poll inside the shell. Do not poll through chat or wait longer without explicit user instruction.
- Codex must take over immediately if delegated work is unfinished or active, including background experiments, jobs, services, downloads, partial implementations, or results needing monitoring or integration.
- After takeover, inspect active processes, workspace changes, outputs, and logs, then monitor or finish the task directly. If waiting expires without recovery, take over; report a blocker only when Codex cannot proceed safely.

## Verification

- Wait for all required calls; for parallel work, integrate only after both finish.
- Prefer the compact final block. Inspect detailed output only on failure or contradiction.
- Verify with exit status, targeted checks, tests, output existence, and Codex's Git review. Inspect detailed diffs only when risk or failed verification warrants it.
- Do not repeat the external model's full reasoning. Do not commit unless requested or clearly required.
