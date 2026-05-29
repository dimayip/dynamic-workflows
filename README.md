# Skills

Public Codex skills by [DannyMac180](https://github.com/DannyMac180).

## Available Skills

- [`codex-dynamic-workflows`](./codex-dynamic-workflows/) - Plan and run supervised Codex-native dynamic workflows with goal mode, subagents or simulated work packets, approval gates, integration, verification, and reusable workflow artifacts.

## Install

Clone this repo and copy a skill folder into your Codex skills directory:

```bash
git clone https://github.com/DannyMac180/skills.git
cd skills
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
cp -R codex-dynamic-workflows "${CODEX_HOME:-$HOME/.codex}/skills/"
```

Then start a new Codex session and invoke it with:

```text
Use $codex-dynamic-workflows to plan and run a supervised multi-agent workflow for this task: ...
```

## License

MIT
