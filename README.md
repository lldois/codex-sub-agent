# Codex Sub Agent

A small, instruction-only Codex skill. It has no scripts, services, state manager, or dependencies.

## Install

Copy the entire `codex-sub-agent` folder to:

```text
~/.agents/skills/codex-sub-agent/
```

The final layout must be:

```text
~/.agents/skills/codex-sub-agent/
├── SKILL.md
└── agents/
    └── openai.yaml
```

Restart Codex only if the skill does not appear automatically.

## Invoke

```text
$codex-sub-agent 分析当前项目的问题，提出方案并完成实现和测试
```

The skill is manual-only because `agents/openai.yaml` sets:

```yaml
policy:
  allow_implicit_invocation: false
```

## Customize the CLI pool

Edit the `CLI pool` section in `SKILL.md`.

Each CLI entry needs only:

- one fixed deep-model command;
- one fixed fast-model command;
- its position in the fallback order.

The default pool is:

1. `agy`
2. `agy1`

Both use `Claude Opus 4.6 Thinking` for deep reasoning and `Gemini 3.6 Flash High` for execution. The two CLIs are treated as conversation-independent. `agy` is preferred until it reaches a clear limit, then the skill falls back to `agy1`.
