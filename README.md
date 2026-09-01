# pr-review-page

An [agent skill](https://agentskills.io) (works with pi, Claude Code, Codex, and any
agent runtime that loads `SKILL.md` folders) that builds a **self-contained interactive
HTML reviewer page** for a GitHub pull request — or for local changes that don't have a
PR yet.

## What it produces

One HTML file, no build step, no external libraries:

- Multi-level inline-SVG diagrams of how the changed components/hooks/APIs interact
  (page → component → hooks → API → RPC)
- The full PR diff in a persistent side pane, one collapsible section per file
- "⚠ CHECK" callouts that embed the exact diff lines to scrutinize, right where the
  claim is made
- Cross-linking: every symbol, file reference, and diagram node opens and scrolls to
  the matching diff hunk

Works from a GitHub PR (URL or number) or from local changes (uncommitted worktree,
feature branch vs base, commit range).

## Install

Copy or clone this folder into your agent's skills directory:

```bash
git clone <this-repo> ~/.agents/skills/pr-review-page
```

Requires an agent with shell + file access. Optional: Playwright or any headless
browser for the built-in verification step.

## Usage

Tell your agent:

- "help me review https://github.com/owner/repo/pull/123"
- "make an overview of this PR as a page"
- "diagram the changes on my branch"
- "review my local changes as a page"

## Contents

- `SKILL.md` — the skill instructions
- `assets/pr-review-page-template.html` — the HTML template the agent fills in

## License

MIT — see [LICENSE](LICENSE).
