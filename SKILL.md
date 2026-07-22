---
name: codex-sub-agent
description: Manual-only supervisor for delegating work to an ordered pool of external AI CLI tools. Use only when explicitly invoked by the user.
---

# Codex Sub Agent

Act as a low-cost supervisor. Let an external CLI perform most substantive reasoning and execution. You route the task, wait synchronously, perform cheap verification, and report the result.

## CLI pool

Try tools in this order. Each tool has independent conversations and quotas.

### 1. `agy`

- Deep model:
  `agy {SESSION} --model "Claude Opus 4.6 Thinking" -p "{PROMPT}" --cwd "{WORKDIR}"`
- Fast model:
  `agy {SESSION} --model "Gemini 3.6 Flash High" -p "{PROMPT}" --cwd "{WORKDIR}"`

Set `{SESSION}` to `-c` when continuing the same task in the same workspace. Omit it for a new, unrelated task.

### 2. `agy1`

- Deep model:
  `agy1 {SESSION} --model "Claude Opus 4.6 Thinking" -p "{PROMPT}" --cwd "{WORKDIR}"`
- Fast model:
  `agy1 {SESSION} --model "Gemini 3.6 Flash High" -p "{PROMPT}" --cwd "{WORKDIR}"`

The model strings must match the stable names shown by each CLI's `/model` picker. If they differ locally, edit only the four command templates above.

To add another CLI, copy an entry below `agy1` and provide that CLI's fixed deep-model and fast-model commands. Keep the desired fallback order.

## Routing

- Do trivial management yourself: concise inspection, known commands, file moves, `git status`, `git diff --stat`, targeted verification, and commits requested by the user.
- Use the deep model for novel ideas, difficult reasoning, architecture, experiment design, ambiguous decisions, difficult diagnosis, and interpreting surprising results.
- Use the fast model for ordinary implementation, code search, information search, debugging, tests, experiments, data processing, and routine edits.
- For a mixed complex task, normally call the deep model once for a concise actionable plan, then call the fast model to implement and verify it.
- Do not split one coherent task into many small delegations. Give one external turn enough scope to finish a meaningful unit of work.
- Reuse the same CLI conversation for a continuing task whenever useful.

## Delegation prompt

Send only the objective, essential constraints, relevant paths, and completion criteria. Tell the external agent to inspect the workspace itself; do not paste large files, logs, diffs, or the whole chat.

Ask it to work autonomously, perform its own checks, and end with at most 12 lines:

```text
STATUS: done | blocked | failed
SUMMARY: brief result
CHANGED: files or none
CHECKS: commands and outcomes
NEXT: none or one required next action
```

For a deep-model planning call, require a plan of at most 250 words so it can be passed cheaply to the fast model.

## Failover

- Run only one external CLI process at a time.
- Start with the first CLI in the pool.
- On a clear quota, rate-limit, unavailable-model, authentication, or executable failure, try the next CLI with the same role.
- Assume a new CLI has no conversation history. Tell it to inspect the current workspace and include only a brief handoff.
- If the failed CLI made partial changes, continue from the current workspace state; do not restart blindly.
- If every CLI is unavailable, complete the task yourself.
- Do not repeatedly retry a CLI that has clearly reached its limit.

## Supervision

- Wait for each CLI command to finish; do not poll it through repeated conversational turns.
- Consume only its compact final response unless it failed or verification contradicts it.
- Verify cheaply with exit status, targeted checks, output existence, `git status --short`, and `git diff --stat`.
- Inspect detailed diffs only where risk or failed verification requires it.
- Do not redo or restate the external model's full reasoning.
- Do not commit unless the user requested a commit or the surrounding task clearly requires one.
