---
name: using-codegraph
description: Use when navigating code in a repo with a .codegraph/ index — orienting in an unfamiliar area, locating a symbol, tracing callers or the blast radius of a change, picking which tests to run, or judging a codegraph_context injection — and before any multi-file Grep/Read sweep.
---

# Using codegraph

## Overview

`codegraph --help` lists the commands. This skill is the routing knowledge the help can't carry: which tool wins for which question, and the places the output is quietly load-bearing. Everything here also applies to the MCP tools (`codegraph_explore`, `codegraph_node`) — same output, different transport.

## The ladder

Stop at the first rung that answers the question:

1. **You hold an exact string** — error message, literal, config key, log line → `rg`. Grep wins on exact strings; use it without ceremony.
2. **You can name the symbol, even roughly** → `codegraph query "<name>"` to locate it, `codegraph node <symbol>` for its source plus caller/callee trail. Fuzzy, so a half-remembered name is enough.
3. **You need orientation** — "how does X work", an unfamiliar subsystem, the start of a feature or bug task → `codegraph explore "<terms>"`. One call returns the relevant symbols' verbatim source, call paths, and blast radius. This replaces the search → read → read → read loop, and replaces it only if you let it (see below).
4. **You're about to edit** — "what breaks if I change this" → `codegraph impact <symbol>`; `callers`/`callees` for one direction only.
5. **You've edited** — which tests cover the damage → `git diff --name-only | codegraph affected --stdin -q` pipes straight into a test runner.

A multi-file Grep/Read sweep is the signal you skipped a rung: name the question, re-enter the ladder.

## Fat calls

`explore` accepts multiple terms — `codegraph explore "parser tokenizer lexer"` is one round-trip; three separate explores are three. Batch the names you already know you need.

## Explore output is completed Reads

`explore` and `node` print **verbatim, line-numbered, on-disk source** — the output says so. Treat each block as a Read you have already performed and go straight to editing or answering. Re-Reading those files is the single move that erases the tool's entire token advantage.

## Judging a codegraph_context injection

Repos with the MCP hook inject a `codegraph_context` block naming symbols that match the prompt. The match is **keyword, not intent** — a prompt containing "build" will surface every `build_*` symbol in the index. Treat the block as a hint: when the named symbols are plausibly what the task touches, one fat `explore` on them is the cheapest possible start; when they're lexical coincidence, enter the ladder as if the block weren't there.

## Trust, staleness, and absence

- Output prefixed with a staleness warning (⚠️) means pending file changes: run `codegraph sync`, then re-run the query. After large git operations (rebase, checkout of an old branch), `sync` first.
- `codegraph status` when results look wrong; `codegraph unlock` when indexing is blocked by a stale lock.
- **No `.codegraph/` directory**: proceed with grep — the ladder's rung 1 covers everything, just at higher token cost — and suggest `codegraph init` to the user once. Degraded results beat a blocked task.

## Programmatic use

`query`, `impact`, `affected` take `-j/--json`; `affected -q` emits bare file paths. Prefer these when the output feeds a script rather than your own reasoning.
