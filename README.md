# Ruflo

**A multi-agent harness for Claude Code and Codex.**

Ruflo is a CLI and MCP server that turns Claude Code into a coordinated team of AI agents instead of one assistant working in a loop. It adds three things on top of a normal Claude Code install:

1. **Swarms of agents** that can split up work and message each other.
2. **Persistent memory** so the model remembers what worked across sessions, not just within one chat.
3. **Hooks** that automatically route tasks to cheaper or more capable models based on what they actually are.

You still write code in Claude Code the way you already do. Ruflo runs in the background.

---

## What problem it solves

A single Claude session forgets everything when it ends, can only do one thing at a time, and uses the same model for "rename a variable" as it does for "design the auth system." Ruflo fixes the three:

- **Forgetting** — every successful pattern is stored in a vector database. Next time you ask something similar, the agent has the prior solution as context.
- **One thing at a time** — instead of one agent looping over a feature, several named agents (architect, coder, tester, reviewer) work in parallel and message each other directly.
- **One model for everything** — a routing layer picks the cheapest tier that can actually do the task. Trivial code transforms run as deterministic codemods at $0; medium tasks go to Haiku (~$0.0002); hard reasoning goes to Sonnet or Opus.

---

## Install

You need Node 20+, a Claude Code install, and an Anthropic API key (or another supported provider).

```bash
# Option 1 — the wrapper (recommended). Sets everything up in the current directory.
npx ruflo init

# Option 2 — the CLI directly.
npx @claude-flow/cli@latest init --wizard

# Option 3 — Claude Code plugin only (no MCP server, slash commands only).
/plugin marketplace add ruvnet/ruflo
/plugin install ruflo-core@ruflo
```

After `init`, restart Claude Code so it picks up the new MCP server and hooks.

To check the install:

```bash
npx ruflo doctor --fix
```

---

## What `init` does

It writes a few things into your project and your Claude config:

| Path | What it is |
|------|------------|
| `.claude/settings.json` | Hooks wiring (pre-edit, post-edit, session-start, etc.) so Ruflo gets called by Claude Code |
| `.claude/helpers/` | Small scripts the hooks point at — pattern recording, routing decisions, statusline |
| `.claude-flow/` | Local state — memory database, learned patterns, session logs |
| `CLAUDE.md` | Project rules Claude Code reads on every turn. Edit this to teach Claude about your project |
| `data/memory/` | The vector-indexed memory store (created on first use) |

No files outside the project directory are touched. To undo, delete those paths.

---

## Three things you can actually use day-to-day

### 1. Spawn a team instead of a single agent

In Claude Code, instead of asking one agent to do everything, ask it to spawn a team:

> "Use a swarm to add OAuth login. Architect → coder → tester → reviewer."

Claude will use the Task tool to spawn named background agents that message each other. You'll see progress for each one. You don't have to manage them; they coordinate via `SendMessage`.

### 2. Search past work

The memory store is semantic — search by meaning, not by keyword:

```bash
npx ruflo memory search --query "authentication patterns"
npx ruflo memory search --query "the bug where login redirected to /undefined"
```

Use this when you suspect you've solved something similar before and don't want to re-derive it.

### 3. Check what the model would cost before spawning

```bash
npx ruflo hooks route --task "rename all snake_case variables to camelCase in src/api/"
```

The router will tell you whether this is a $0 codemod (it is), a Haiku job, or needs Sonnet/Opus.

---

## Core concepts in one sentence each

| Concept | What it is |
|---------|------------|
| **Agent** | A specialized Claude session with a role (e.g. `coder`, `tester`, `security-architect`) and a system prompt tuned for that role |
| **Swarm** | A group of agents working on the same task with a topology (hierarchical, mesh, hybrid) that controls who can talk to whom |
| **Hooks** | Shell commands Claude Code runs on lifecycle events (before/after edits, before/after commands, on session start). Ruflo uses them to record outcomes and route tasks |
| **Memory** | A vector database (HNSW-indexed) that stores successful patterns and lets future runs retrieve them by semantic similarity |
| **Routing** | A three-tier decision (codemod → Haiku → Sonnet/Opus) that picks the cheapest model that can still do the task |
| **MCP server** | The interface Claude Code uses to call Ruflo's tools (memory, swarm, hooks). Ruflo registers it during `init` |
| **Plugin** | An optional add-on (browser automation, GitHub workflows, neural trading, etc.) — install only what you need |

---

## Common questions

**Do I have to learn all 314 MCP tools and 26 CLI commands?**
No. After `init`, just use Claude Code normally. The hooks and routing run in the background.

**Will it slow Claude down?**
Hooks add a few hundred milliseconds per tool call for the recording path. The routing layer typically saves more time than it costs because trivial tasks skip the LLM entirely.

**Does it work with Codex?**
Yes. `@claude-flow/codex` runs Codex workers in parallel with Claude workers, sharing the same memory namespace. See `docs/dual-mode.md` if you want both.

**Does it phone home?**
No. Memory, hooks, and routing run locally. The only network calls are to the LLM provider you configured (Anthropic, OpenAI, etc.). The optional plugin registry uses IPFS via Pinata.

**Where's my data?**
`.claude-flow/data/memory/` in the project root. SQLite + vector index. Delete the folder to wipe it.

**How do I uninstall?**
Delete `.claude/`, `.claude-flow/`, `CLAUDE.md`, and (if you used the wrapper) `npm uninstall -g ruflo`.

---

## When you should NOT use Ruflo

- You're working in one file for ten minutes. Just use Claude Code directly.
- You don't want anything in your project directory. Use the Claude Code plugins instead (`/plugin install ruflo-core@ruflo`).
- You're on a machine where you can't run a background daemon (the daemon is optional, but several features need it).

---

## Help

| | |
|---|---|
| Docs | https://github.com/ruvnet/ruflo/tree/main/docs |
| Issues | https://github.com/ruvnet/ruflo/issues |
| Releases | https://github.com/ruvnet/ruflo/releases |
| npm — `ruflo` | https://www.npmjs.com/package/ruflo |
| npm — `@claude-flow/cli` | https://www.npmjs.com/package/@claude-flow/cli |
| Cognitum — the agentic engine | https://cognitum.one/ |

To file a bug, run `npx ruflo doctor` and paste the output into the issue. That captures the version, Node version, MCP server state, and which hooks are wired — most of what someone debugging will ask for.

---

## License

MIT. See [LICENSE](LICENSE).

> Ruflo is the rename of Claude Flow. The name is `rUv` ("Ru") + "flow" ("flo"). It's powered by [Cognitum.One](https://cognitum.one/) — a Rust-based engine for embeddings, memory, and the plugin system.
