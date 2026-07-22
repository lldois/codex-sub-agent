# Codex Sub Agent

A small, instruction-only Codex skill for manually delegating work to an ordered pool of external AI CLI tools.

## Install

In Codex, run:

```text
$skill-installer install https://github.com/lldois/codex-sub-agent
```

Then invoke:

```text
$codex-sub-agent 分析当前项目的问题，提出方案并完成实现和测试
```

## Repository layout

```text
codex-sub-agent/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── README.md
```

The skill is manual-only because `agents/openai.yaml` sets:

```yaml
policy:
  allow_implicit_invocation: false
```

## Default CLI pool

1. `agy`
2. `agy2`

Both expose the same model slugs and use:

- Deep reasoning: `claude-opus-4-6-thinking`
- Routine execution: `gemini-3.6-flash-high`

Examples:

```bash
agy --model claude-opus-4-6-thinking
agy --model gemini-3.6-flash-high
agy2 --model claude-opus-4-6-thinking
agy2 --model gemini-3.6-flash-high
```

The two CLIs are conversation-independent. The skill prefers `agy`; after a clear quota or availability failure, it falls back to `agy2`. If both are unavailable, Codex completes the task directly.

## Customize

Edit the CLI pool in `SKILL.md`. Each entry needs a fixed deep-model command, a fixed fast-model command, and its fallback position.
