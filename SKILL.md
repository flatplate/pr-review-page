---
name: pr-review-page
description: >
  Build a self-contained interactive HTML reviewer page for a GitHub pull request: multi-level
  diagrams of how the changed components/hooks/APIs interact, the full PR diff in a persistent
  side pane, "⚠ CHECK" callouts that embed the exact diff lines to scrutinize, and cross-links
  from every symbol, diagram node, and file reference to its diff hunk. Works from a GitHub PR
  (by URL or number) or from LOCAL changes (uncommitted worktree, a feature branch vs its
  base, a commit range) — the page only needs a unified diff plus enough surrounding code to
  explain it. Use when the user asks to review a PR "with a page", "make a review page/diagram
  for PR <url>", "help me review <PR-url>", "make an overview/deep-dive of this PR", "diagram
  the changes on this branch / my working tree", or asks for a diagram of a PR's or branch's
  components and data flow — even if they only say "diagram this PR", "review my local
  changes", "help me review this PR as an html page", or "what did I change here, show me as a
  page". Also use when asked to update or extend an existing review page.
---

# PR Review Page

Build a self-contained HTML page that gives a reviewer a complete mental map of a code
change — a GitHub pull request or local changes (uncommitted worktree, feature branch vs
base, commit range): what the change is, which page/feature it touches, how the changed
pieces call each other, where the data comes from, and — inline, right where the claim is
made — the diff lines that prove each "pay attention to this" claim. The page must be
self-sufficient: a reviewer should never need GitHub (or anything else) open to understand
the change.

The finished artifact is ONE html file. Interactive behaviors (cross-link scrolling, diff
expansion, diagram clicks) run on vanilla JS embedded in the file — no external libraries,
no build step, no server.

## Workflow

### 1. Gather the diff and material

Determine the review source first — the page needs a unified diff plus metadata; everything
else in the workflow is identical for both.

**GitHub PR** (URL or number given):
1. Prefer the repo's own PR-context script when one exists (a wrapper that returns
   bounded PR metadata, files, commits, reviews, checks, and the full diff as JSON —
   check the repo's scripts/docs directory for one). Cap the diff at ~200k chars
   (`--include-diff --max-diff-chars 200000 --json` or equivalent).
   Otherwise fall back to plain GitHub CLI: `gh pr view` + `gh pr diff` /
   `gh api repos/OWNER/REPO/pulls/NN`.
2. Header data comes from the PR: number, title, base/head branch, body summary,
   feature-flag notes.

**Local changes** (no PR — user says "review my branch/working tree", "before I open a PR", or
the changes only exist locally):
1. Detect the range: staged+unstaged → `git diff HEAD` (+ `git status` for untracked files,
   which plain diff misses — list them in the inventory as NEW with no diff hunks);
   a feature branch → `git diff <base>...HEAD` with base = the repo's trunk (main/master
   discovered via `git remote show origin` or the merge-base); a commit or range the user
   names → `git diff <from>..<to>`.
2. Save that diff to a temp file (e.g. `/tmp/local-review.diff`) — the side pane and every
   excerpt come from it.
3. The header adapts: no PR number/chips — use branch (or "working tree") + commit range +
   a one-line `git log` summary instead of the PR body; "View full diff on GitHub" link
   becomes a note like "uncommitted — no remote diff" (or a compare-URL if a base branch
   exists and is pushed).

In both cases:
1. Extract the changed-file inventory (path, status, +/- counts) from the diff.
2. For each file the page will discuss in depth, stage the post-change version
   (`git show <ref>:<path> > /tmp/<name>`, or just copy the working-tree file for local
   changes) so code excerpts come from the changed state, not the base.
3. Read enough surrounding code (the changed files' callers, the page/component that hosts
   them) to explain the change in context. The narrative must start from where the change
   lives in the app and work down — reviewers think top-down: page → feature → hooks → API.

### 2. Build the page from the template

Start from the bundled template (`assets/pr-review-page-template.html`, same directory as
this file), never from scratch — it carries the two-pane layout, the diff-pane JS, and
HOW-TO-USE comments. Read it fully before writing. Copy it to the output path (kebab-case,
next to the repo or in the CWD) and fill it in.

Page structure, in render order:

1. **Header** — PR number or branch/"working tree" + commit range, title/summary,
   one-paragraph what-changed, feature-flag note if
   any, badge legend (NEW / MODIFIED / REWRITTEN / DELETED / UNCHANGED).
2. **Reading-model strip** — one line: narrative → attention callouts contain the exact diff
   lines to scrutinize; click any symbol or diagram node to open it in the full diff pane.
3. **Diagram A — the page**: the product page/tab the PR touches, its existing components,
   what each does, and which parts are this PR's scope (mark them; leave the rest as context).
4. **Diagram B — inside the changed component** (hub-and-spoke): the orchestrating component
   as hub, each child/hook/state as a spoke with its job in one line, plus the reset/race
   rules that govern the state.
5. **Diagram C — data flow**: hooks → API client layer → RPCs, with React Query keys and the
   pagination/aggregation behavior annotated.
6. **Diagram D — shared-code ripple**: what the change ripples into outside the card
   (shared components, moved helpers, changed prop semantics).
7. **Per-area narrative sections** with code excerpts (real, cited file+lines, trimmed to
   what matters) and **attention callouts** (see below).
8. **File inventory** — every changed file with badges and one-line role.

### 3. Attention callouts (the core principle)

Wherever the page says "check this / pay attention to this", show the code right there —
do not just link to it. Each callout:

- Accent left border + "⚠ CHECK" eyebrow + one-line claim of what to verify.
- Immediately below: a mini diff excerpt (2–15 lines, verbatim from the PR diff, green/red
  gutters, `@@` header) of exactly the lines that prove the point. If the claim has no diff
  (pre-existing context code), show a plain code excerpt with file+lines cited.
- Footer citing file + hunk, plus a cross-link into the full diff pane.

Only the lines that prove the point — no full files in callouts. Hunt for these claims in
reset effects, batched state updates, disabled-query guards, pagination loops, dedup/fallback
branches, NaN/overflow guards, query-key mirroring, and error-path handling. Every callout's
claim must be checkable against the diff excerpt it shows.

### 4. Diagrams

Inline SVG, one per area (3–4 total). Rules that made the first version good:

- Nodes carry file/function name + one-line role; badges for NEW/MODIFIED/DELETED (deleted =
  faded/strikethrough but present, so the reviewer sees what disappeared).
- Layered vertical flow (page → components → hooks/helpers → API → RPC) with the main
  component as hub inside its band; orthogonal connectors with masked labels.
- Density may exceed a normal diagram's budget here — this is a scrollable reviewer document,
  not a slide. Tall canvas is fine.
- All diagram nodes that correspond to changed files are clickable: same cross-link behavior
  as text links (open + scroll the diff pane + flash the hunk). `tabindex=0`, Enter/Space
  handled, `cursor: pointer`, hover stroke. Add a "click nodes to open the diff" hint under
  each diagram.

### 5. Full-diff side pane

- Sticky right pane, viewport height, independent scroll; one collapsible `<details>` per
  file (path + badge + +N/−N in the summary), first file open, rest collapsed; verbatim diff.
- Hunk-level ids: `h-<sanitized-path>-<newLine>`. A file index with add/del counts at the top
  of the pane and a GitHub "view full diff" link (omit or replace with a compare-URL/note for
  purely local changes).
- Code in the pane at ≥12.5px, line-height ~1.55 (smaller reads as decoration, not reference);
  per-file horizontal scroll for long lines.
- Cross-linking (vanilla JS): every prose symbol, file reference, and diagram node links via
  `data-file`/`data-hunk`; clicking expands the file's `<details>`, smooth-scrolls the pane
  (page must not jump), flashes the hunk briefly. Curate link→hunk mappings per reference —
  don't dump every symbol onto the same file header.
- Narrow fallback: below ~1360px the pane becomes a toggleable drawer with a Show/Hide button
  and a close button.
- `scroll-behavior: smooth` on `html` and the pane.

### 6. Verify before handing over

Headless-browser check (Playwright or equivalent), all pass before done:

- Every cross-link and diagram-node link resolves to an existing file/hunk anchor; clicking
  expands, scrolls, flashes.
- No JS console errors.
- No horizontal overflow at ~1720px, ~1500px, ~1000px.
- Diff pane code ≥12.5px.
- Open the file in the browser as the last action.

## Optional: publishing behind auth

GitHub PR bodies cannot host interactive HTML (sanitizer strips script/style/SVG handlers),
and public CDNs (jsDelivr) only serve public repos. When the user wants reviewers to open the
page from a link, offer internal-only options in this order:

1. **GitHub Pages on the private repo** (enterprise plan → org-members-only URLs). Check
   `gh api repos/<org>/<repo>/pages` — `build_type: workflow` + private repo is the signal.
   Extend the repo's existing pages workflow or add a minimal deploy step.
2. **An existing internal static host** (e.g. a builds/CDN bucket the repo's CI already
   publishes to) — needs a discovered upload path; ask the user for the bucket/workflow.
3. Never publish to a public URL. Never push to the PR branch or edit the PR body without
   the user's explicit go-ahead.

## Diagnosing weak output

If a produced page feels like a generic architecture sketch: the gather step (1) was thin —
the narrative depth of this page comes from reading the surrounding code (parent page,
sibling features, store) before writing, not from the diff alone. Re-gather, then rebuild.
