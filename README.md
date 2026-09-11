# mi-coding-agent

**Maximum Impact Studio's portable coding-agent skills.** Three battle-tested skills for
the GitHub PR loop — open a review-ready PR, review one relentlessly until it's clean,
and address reviewer feedback round by round — packaged so any agent runtime can pick
them up directly, or a brand-new agent can bootstrap from this repo alone.

Built and refined against a real production codebase (a mobile game with a long-running,
adversarial review process), then generalized here to be project-agnostic.

- 🚀 **`open-pr`** — build a change in its own worktree, prove it with mutation testing,
  self-audit before pushing, and open a PR whose body states what it does *not* prove.
- 🔍 **`review-pr`** — review a PR against its *entire* history (not just the diff since
  last time), post one consolidated comment, and watch for the author's response —
  bounded convergence keeps a long review from drifting into infinite nitpicking.
- 🛠️ **`address-review-comments`** — fix the latest reviewer batch, self-audit
  (consistency / regression / staleness / platform-fact-check), commit, push, reply
  point-by-point. Loop-aware: no-ops on nothing new, stops clean on an all-clear.

All three skills follow the [AgentSkills open standard](https://agentskills.io) — a
`SKILL.md` with YAML frontmatter plus a markdown body — so they load natively in
**OpenClaw**, **Claude Code**, **Codex**, **Hermes**, and any other AgentSkills-compatible
runtime, with per-runtime notes where the underlying mechanism differs (see
`skills/WAITING.md` for how the "watch a PR for a response" loop maps to each host's
actual background/scheduling primitive).

## What's in the repo

```text
mi-coding-agent/
  skills/
    open-pr/
      SKILL.md
      agents/openai.yaml          # Codex display metadata
      .claude-plugin/plugin.json  # Claude Code plugin manifest
    review-pr/
      SKILL.md
      agents/openai.yaml
      .claude-plugin/plugin.json
    address-review-comments/
      SKILL.md
      agents/openai.yaml
      .claude-plugin/plugin.json
    WAITING.md                    # shared "watch a PR without wasting tokens" contract
  .claude-plugin/marketplace.json # Claude Code marketplace catalog for this repo
  LICENSE                         # MIT
```

## Bootstrap a brand-new agent from this repo

Clone it, then point any AgentSkills-compatible runtime at the `skills/` directory (or a
specific skill inside it) as its skills root. No build step, no dependencies beyond
`gh`, `git`, and `jq` on the machine actually running the agent.

```bash
git clone https://github.com/arunjeetsingh/mi-coding-agent.git
```

From there, follow whichever section below matches the runtime you're bootstrapping.

## Add to an existing agent

### OpenClaw

```bash
# Into the active workspace's skills/ (visible to that agent only)
openclaw skills install git:arunjeetsingh/mi-coding-agent

# Or shared across every local agent
openclaw skills install git:arunjeetsingh/mi-coding-agent --global
```

`openclaw skills install git:owner/repo[@ref]` clones the repo and looks for `SKILL.md` at
the source root — this repo groups its three skills under `skills/`, and OpenClaw
discovers a `SKILL.md` anywhere up to 6 levels under a configured skill root, so once
cloned into a skills directory (or added via `skills.load.extraDirs` pointing at a local
checkout's `skills/` folder) all three (`open-pr`, `review-pr`, `address-review-comments`)
become available. Verify with:

```bash
openclaw skills list
```

Once these are published to ClawHub (see below), install with:

```bash
openclaw skills install @maximum-impact-studio/open-pr
openclaw skills install @maximum-impact-studio/review-pr
openclaw skills install @maximum-impact-studio/address-review-comments
```

### Claude Code

Add this repo as a plugin marketplace, then install whichever skills you want:

```text
/plugin marketplace add arunjeetsingh/mi-coding-agent
/plugin install open-pr@mi-coding-agent
/plugin install review-pr@mi-coding-agent
/plugin install address-review-comments@mi-coding-agent
```

Restart Claude Code if it doesn't pick up the new skills immediately.

### Codex

Codex reads skills from `$CODEX_HOME/skills` (defaults to `~/.codex/skills`). Copy or
symlink the ones you want:

```bash
git clone https://github.com/arunjeetsingh/mi-coding-agent.git /tmp/mi-coding-agent
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
cp -r /tmp/mi-coding-agent/skills/open-pr "${CODEX_HOME:-$HOME/.codex}/skills/"
cp -r /tmp/mi-coding-agent/skills/review-pr "${CODEX_HOME:-$HOME/.codex}/skills/"
cp -r /tmp/mi-coding-agent/skills/address-review-comments "${CODEX_HOME:-$HOME/.codex}/skills/"
cp /tmp/mi-coding-agent/skills/WAITING.md "${CODEX_HOME:-$HOME/.codex}/skills/"
```

Each skill's `agents/openai.yaml` carries the Codex-facing display name and default
prompt. Where a skill's body mentions Claude Code's `Monitor`/`TaskStop`, use the Codex
heartbeat lifecycle documented in `skills/WAITING.md` instead — that file is the shared
contract all three skills point to for the "watch a PR without burning tokens" loop, and
it spells out the Codex-specific mechanics (`automation_update` heartbeats, ownership
markers, reconciliation) alongside the OpenClaw and Claude Code equivalents.

### Hermes

Hermes loads skills from `~/.hermes/skills/` (or an external directory merged in via
`skills.external_dirs` in `config.yaml`) and is explicitly compatible with the
[agentskills.io](https://agentskills.io) standard these skills follow:

```bash
git clone https://github.com/arunjeetsingh/mi-coding-agent.git /tmp/mi-coding-agent
mkdir -p ~/.hermes/skills
cp -r /tmp/mi-coding-agent/skills/open-pr ~/.hermes/skills/
cp -r /tmp/mi-coding-agent/skills/review-pr ~/.hermes/skills/
cp -r /tmp/mi-coding-agent/skills/address-review-comments ~/.hermes/skills/
cp /tmp/mi-coding-agent/skills/WAITING.md ~/.hermes/skills/
```

Or point `config.yaml`'s `skills.external_dirs` at a checkout's `skills/` directory to
pick up updates on `git pull` without recopying.

## Requirements

All three skills assume a Unix-like shell with `git`, `gh` (GitHub CLI, authenticated),
and `jq` on `PATH`. `review-pr` and `address-review-comments` also expect the repo you're
running them against to be a `gh`-authenticated GitHub repository.

## Standing rules across all three skills

- **Never merge, never deploy.** These skills open PRs and reply to reviews; a human
  merges and a human deploys. `gh pr merge` is never called.
- **Never print secrets** in a file, commit, or comment.
- **One commit per review round.** `address-review-comments` never mixes a partial edit
  with a commit.
- Every claim about a third-party platform (a store's release states, an API's allowed
  transitions, a CLI's actual flags) gets fact-checked before it ships in a PR body or
  review comment — a wrong platform fact is treated as a blocker, not a nitpick.

## Contributing

Public repo, MIT licensed, PRs welcome. If you use these skills against a different kind
of repo (a library instead of a game, a monorepo, a different CI setup) and hit an edge
case the skills don't handle, open an issue or a PR — the goal is for these to stay
project-agnostic.

## License

[MIT](./LICENSE) — the most permissive common open-source license, chosen specifically to
make it as easy as possible for individuals *and* companies to fork, adapt, and contribute
back.
