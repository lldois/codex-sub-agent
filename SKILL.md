---
name: codex-sub-agent
description: Manually supervise and delegate work to an ordered pool of external AI CLI tools. Use only when the user explicitly invokes $codex-sub-agent; send difficult reasoning to a fixed deep model, routine implementation to a fixed fast model, preserve useful conversations for continuing work, allow safe parallel use of independent CLIs, recover from temporary quota outages, reserve all Git mutations for Codex, and keep Codex focused on low-cost supervision and verification.
---

# Codex Sub Agent

Act as a low-cost supervisor. Delegate most substantive reasoning and execution to external CLIs, wait for useful work to finish, retain exclusive control over Git mutations, perform cheap verification, and report the result.

## CLI pool

Use these CLIs in priority order. Their conversations and quotas are independent.

Run each command from the target workspace. Do not pass a `--cwd` flag to AGY; set the shell working directory or change directory first.

### 1. `agy`

- Deep model:
  `agy {SESSION} --model claude-opus-4-6-thinking -p "{PROMPT}"`
- Fast model:
  `agy {SESSION} --model gemini-3.6-flash-high -p "{PROMPT}"`

### 2. `agy2`

- Deep model:
  `agy2 {SESSION} --model claude-opus-4-6-thinking -p "{PROMPT}"`
- Fast model:
  `agy2 {SESSION} --model gemini-3.6-flash-high -p "{PROMPT}"`

Use safe shell quoting for multiline prompts and metacharacters. Never concatenate untrusted prompt text into a shell command without proper escaping.

To add another CLI, copy an entry below `agy2`, provide fixed deep-model and fast-model commands, and preserve the desired priority order.

## Preserve conversations

- Treat one CLI conversation as sticky to one continuing workstream.
- Set `{SESSION}` to `--continue` only when continuing the same workstream with the same CLI in the same workspace. Omit it for a new or unrelated task.
- Prefer keeping a continuing task on the same healthy CLI and in the same conversation. Do not switch merely to rebalance load or because another CLI has recovered.
- For a mixed task, prefer using the same CLI conversation for the deep-model planning step and the fast-model implementation step when continuing context is useful.
- After switching to another CLI, omit `--continue` on its first call and provide a short handoff. Later calls to that CLI for the same workstream may use `--continue`.
- Because CLI conversations are independent, never assume one CLI can see another CLI's hidden history. Use the workspace and a concise handoff as the transfer boundary.
- If a long task reaches a natural milestone, keep the current conversation for the next related milestone unless its context has become polluted or the workstream has materially changed.

## Keep Git mutations under Codex control

- Codex exclusively owns all Git mutations and Git decisions.
- External agents may run read-only Git inspection commands when useful, including `git status`, `git diff`, `git log`, `git show`, `git branch --show-current`, `git branch --list`, `git tag --list`, and `git worktree list`.
- External agents must not run any Git command that can change repository, index, branch, ref, remote, stash, submodule, or worktree state. Prohibited commands include `git add`, `git commit`, `git checkout`, `git switch`, `git restore`, `git reset`, `git revert`, `git clean`, `git merge`, `git rebase`, `git stash`, `git pull`, `git fetch`, `git push`, `git tag` creation or deletion, `git branch` creation or deletion, `git worktree add/remove/move/prune`, and submodule update commands.
- External agents must not modify `.git`, Git hooks, Git configuration, index files, refs, branches, commits, stashes, submodules, worktree metadata, or remote configuration.
- When a Git command is not clearly read-only, the external agent must not run it.
- Codex checks the baseline Git state before delegation and inspects the resulting Git state after each agent finishes.
- Codex alone creates or removes branches and worktrees, assigns isolated directories, stages files, resolves conflicts, reverts changes, commits, tags, fetches, pulls, and pushes.
- External agents may use read-only history and diff inspection to understand the task, but Codex remains responsible for final diff review and every Git-changing action.
- Every delegation prompt that permits workspace access must explicitly say: `Read-only Git inspection is allowed. Do not run Git commands that change repository state; Codex manages all Git mutations.`

## Route work

- Perform management directly: all Git-changing operations, concise inspection, known commands, file moves, targeted verification, and user-requested commits.
- Use the deep model for novel ideas, difficult reasoning, architecture, experiment design, ambiguous decisions, difficult diagnosis, and interpretation of surprising results.
- Use the fast model for ordinary implementation, code search, information search, debugging, tests, experiments, data processing, and routine edits.
- For a mixed complex task, normally call the deep model once for a concise actionable plan, then call the fast model to implement and verify it.
- Do not split one coherent task into many small delegations. Give one external turn enough scope to finish a meaningful unit of work.

## Use CLIs in parallel safely

- Never run more than two external CLI processes at the same time, regardless of how many CLIs are configured. A third process must wait until one active process exits.
- Default to one external CLI process. Use two only when the subtasks are independent, parallelism clearly reduces elapsed time, and the machine has enough available memory.
- If either task is memory-heavy or the machine is already under memory pressure, serialize the work instead of using both slots.
- Different CLIs may run at the same time when their subtasks are independent and parallelism clearly reduces elapsed time.
- Safe examples include two read-only investigations, separate repositories, Codex-created separate worktrees, or disjoint file scopes with no shared generated state.
- Do not run dependent stages in parallel. Planning that must guide implementation finishes before implementation starts.
- Do not let two agents write overlapping files, share one mutable output directory, or run conflicting experiments at the same time.
- When both agents must write, Codex must first isolate them in separate worktrees or clearly disjoint directories; otherwise serialize them. Agents may inspect worktree information read-only, but only Codex may create, remove, move, or prune worktrees.
- Do not run two concurrent processes from the same CLI when both would rely on that CLI's current `--continue` conversation. Use at most one active continuing process per CLI.
- Give each parallel agent a precise scope and completion criterion. Codex merges, compares, and validates their results only after both finish.

## Compose delegation prompts

Send only the objective, essential constraints, relevant paths, assigned scope, and completion criteria. Tell the external agent to inspect ordinary workspace files itself. Do not paste large files, logs, diffs, or the whole chat.

Every prompt that permits workspace access must include:

```text
Read-only Git inspection is allowed. Do not run Git commands that change repository state; Codex manages all Git mutations.
```

Ask the agent to work autonomously, perform its own checks, and end with at most 12 lines:

```text
STATUS: done | blocked | failed
SUMMARY: brief result
CHANGED: files or none
CHECKS: commands and outcomes; any Git commands must be read-only
NEXT: none or one required next action
```

For a deep-model planning call, require a plan of at most 250 words so it can be passed cheaply to the fast model when needed.

## Handle quota and temporary unavailability

- Start a new workstream with `agy` unless it is already known to be unavailable. Use `agy2` after a clear quota, rate-limit, unavailable-model, authentication, or executable failure.
- If the current workstream is already progressing successfully on `agy2`, do not move it back to `agy` merely because `agy` later recovers. Preserve the active conversation and switch only at a natural boundary or after failure.
- A recovered higher-priority CLI may be used for a new independent workstream.
- If all CLIs are currently unavailable, choose between direct takeover and bounded waiting according to urgency, expected recovery time, and how costly the task is for Codex to complete itself.
- For urgent, small, or mechanically manageable work, Codex should take over directly.
- For non-urgent work where quota recovery is likely soon, wait without consuming conversational turns, then retry. Use a small bounded window rather than an unbounded loop; when the user gives no budget, a reasonable default is up to 15 minutes with retries about every 5 minutes.
- Perform waiting inside the shell process when possible. Do not repeatedly narrate or poll through chat messages.
- After the wait budget expires, either take over directly if practical or report the temporary blocker instead of retrying indefinitely.
- Do not repeatedly retry a CLI that has clearly reached a long-duration or daily limit.

## Supervise and verify

- Before delegation, Codex records the relevant Git baseline and checks for pre-existing user changes.
- Wait for each required CLI command to finish. For parallel calls, wait for all required independent subtasks before integrating results.
- Keep full CLI output out of the Codex conversation when possible. Consume the compact final block; inspect only the tail of detailed output on failure or contradiction.
- External agents may use read-only Git inspection during their work, but Codex performs the authoritative post-task `git status`, diff review, and Git-state validation.
- Verify behavior with exit status, targeted checks, tests, and output existence.
- Inspect detailed diffs only when risk or failed verification requires it.
- Do not redo or restate the external model's full reasoning.
- Do not commit unless the user requested a commit or the surrounding task clearly requires one.
