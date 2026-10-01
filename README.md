# Liora Labs Claude Code plugins

The plugin marketplace for [Liora Labs](https://github.com/LioraLabs) tools.

```bash
claude plugin marketplace add LioraLabs/claude-plugins
```

## Plugins

### cliban

Workflow skills for [cliban](https://github.com/LioraLabs/cliban), the
self-hosted kanban board your agents can't forget: issue and bug tracking,
project status, ticket capture, progressive project memory, the Linear
bridge (`import` an issue, work it, `push` it back), and
`complete-milestone`, which drives every issue in a milestone through its
own agent in dependency order. Ships `/bugs` and `/status` commands.

```bash
claude plugin install cliban@lioralabs
```

The plugin's canonical source lives in the cliban repo itself
([`LioraLabs/cliban/plugin/`](https://github.com/LioraLabs/cliban/tree/main/plugin)),
so skill updates ship with the CLI they document.

### cook

Skills for [cook](https://github.com/LioraLabs/cook), the build system that
makes artifacts just like grandma used to: using cook day to day, authoring
Cookfiles, and authoring cook modules.

```bash
claude plugin install cook@lioralabs
```

### codegraph

A discipline layer for [codegraph](https://github.com/colbymchenry/codegraph),
the pre-indexed code knowledge graph: a routing ladder that sends agents to
the graph for orientation, call tracing, and blast radius — and to grep when
grep honestly wins. The tool itself installs separately; this plugin teaches
agents when to reach for it.

```bash
claude plugin install codegraph@lioralabs
```

### waytchme

Make [WaytchMe](https://github.com/LioraLabs/waytchme) screen recordings.
`script-video` reads a codebase and writes the video script: named scenes,
each with on-screen actions and voice-over lines, plus the vocabulary for
transcription. `edit-take` makes the agent Murphy, the editor you talk to
while recording: it obeys spoken directives, cuts fluff, and hands you the
review before anything renders. Needs a WaytchMe checkout.

```bash
claude plugin install waytchme@lioralabs
```

### ppu-toys

Create uploadable [ppu.toys](https://ppu.toys) demos with the standalone
`ppu` CLI: Lua register control, Mode 7, HDMA, PNG import, native rendering,
and playback/seek checks. Includes the `creating-demos` skill. Install
`ppu` 0.1.0 or newer separately; no development repositories are needed.

```bash
claude plugin install ppu-toys@lioralabs
```

## License

MIT. See [LICENSE](LICENSE).

### game-bible

House rules for building games with agents without giving up control. The
`game-bible` skill keeps every value a designer might tweak in readable KDL
and CSS (the tweak map: nothing tweakable hides in source), makes agents build
the game's vocabulary before its content, and uses Storybook as the game
editor. `game-bible-tidy` audits a game repo that has drifted and fixes it
without changing how it plays. It ships with a small working starter game
that every new game begins as a copy of.
[Read how it was developed](https://lioralabs.dev/blog/agentic-speed-human-control).

```bash
claude plugin install game-bible@lioralabs
```
