# Liora Labs Claude Code plugins

The plugin marketplace for [Liora Labs](https://github.com/LioraLabs) tools.

```bash
claude plugin marketplace add LioraLabs/claude-plugins
```

## Plugins

### cliban

Workflow skills for [cliban](https://github.com/LioraLabs/cliban), the
self-hosted, AI-agent-first kanban board: issue and bug tracking, project
status, ticket capture, progressive project memory, and `complete-milestone`
— orchestration that drives every issue in a milestone through its own agent
in dependency order. Ships `/bugs` and `/status` commands.

```bash
claude plugin install cliban@lioralabs
```

The plugin's canonical source lives in the cliban repo itself
([`LioraLabs/cliban/plugin/`](https://github.com/LioraLabs/cliban/tree/main/plugin)),
so skill updates ship with the CLI they document.

### cook

Skills for [cook](https://github.com/LioraLabs/cook), the artifact-aware
build system: using cook day-to-day, authoring Cookfiles, and authoring cook
modules.

```bash
claude plugin install cook@lioralabs
```

## License

MIT — see [`LICENSE`](LICENSE).
