# Skill: Story Review HTML Companion

*Process/tooling skill — a reusable recipe for AI-assisted MoonBlokz work, not domain knowledge. Non-authoritative: it prescribes how to build a review aid, not any MoonBlokz fact.*

## Purpose

Produce a single self-contained, navigable HTML page that makes the deep review of one implemented story tractable. Raw diffs are hard to review deeply; this page reframes a story's changes as **per-file, per-method** units, each preceded by a plain-language explanation and shown as a whole-method git-diff, with hover tooltips on changed lines and a set of orientation aids (diagrams, tables, a reviewer checklist). Publish it as a private Artifact and hand the reviewer the link.

Use it after a story's code is committed (typically on its branch or merged), when the reviewer wants to understand *what changed and why* before reading the source line by line.

## Inputs

- The story's diff range: `BASE..HEAD` (e.g. the pre-story merge base to the merged commit, or `main..<story-branch>`), scoped to source (`-- src/`).
- The story specification file (BMAD story `.md`) for its Acceptance Criteria, Dev Notes, and any review findings.
- The changed source files, read in full for surrounding context.
- The owning subsystem's PRD/architecture docs for FR/section citations.

## Step 1 — Gather accurate diff data

Drive everything from the real diff; never paraphrase code from memory.

- `git diff --stat BASE..HEAD -- src/` for the file list and +/- counts.
- `git diff -W BASE..HEAD -- <file>` per file: `-W` (`--function-context`) expands each hunk to the whole enclosing function, which is exactly the "show the whole method" requirement.
- For a method that is *modified* (not purely added), also take a tight `git diff BASE..HEAD` hunk to see the precise +/- lines.
- Classify each change unit: **added** (new method/enum/module — whole thing is `+`), **modified** (existing method — mixed context/`+`/`-`), or **moved** (removed from one file, added in another).
- Read the final source around each unit so the shown method body is exact and the explanation is grounded.

## Step 2 — Page structure

Left sticky sidebar (file-grouped navigation with a scrollspy active-state) + a scrolling main column. Order the main column so the reviewer builds a model before hitting raw diffs:

1. **Header + stat chips** — story title, `BASE`/`HEAD`, and chips: files, +/− lines, test count (before → after), clippy (new warnings), new `unsafe`, gated/API count. A short "reading order that works" callout.
2. **Orientation diagrams** — a state machine and/or a control-flow diagram that captures the story's spine (see Step 5). Only include diagrams that encode something true about the change.
3. **API surface table** — new/changed public methods: signature, return type, kind, what this story built vs deferred, owning epic.
4. **AC ↔ code ↔ test table** — one row per Acceptance Criterion: what it requires, where it lives, which test covers it. This is the traceability spine of the review.
5. **Code-review findings panel** — if an adversarial/code review ran: severity, finding, resolution (fixed / accepted / deferred).
6. **Per-file sections** — each file gets a heading with its +/- badge and a one-line role, then one **change card** per unit (Step 3).
7. **Tests table** — every added test: name, what it asserts, which AC.
8. **Reviewer checklist** — the specific things this story needs a human to confirm (with tickable, strike-through checkboxes).

**Reconcile the raw line count.** The +/- stat chip is the raw `git diff` count, but the page shows only a fraction of it as code (doc-comments elided, tests tabulated, repetitive patterns shown once) — a gap that confuses reviewers if left implicit. Next to the stat, add a small breakdown (a segmented bar + legend) that splits the added total into **production logic / doc-comments / test code / comments+blank+attributes**, and state that only the production-logic slice is rendered as diffs. Doc-heavy crates (like `moonblokz-blockchain`, where rustdoc + tests routinely dominate a diff) make this the difference between a confusing "+658 but I only see ~100 lines" and an understood one. Compute the split with an `awk` pass over the diff:

```
git diff BASE..HEAD -- src/ | awk '
  /^diff --git/{intest=0} /mod tests/{intest=1} /^@@/{next}
  /^\+/&&!/^\+\+\+/{tot++;
    if(intest){tests++;next}
    if($0~/^\+[[:space:]]*(\/\/\/|\/\/!)/)doc++;
    else if($0~/^\+[[:space:]]*\/\//)c++;
    else if($0~/^\+[[:space:]]*$/)b++;
    else if($0~/^\+[[:space:]]*#\[/)a++;
    else code++}
  END{printf "total=%d logic=%d doc=%d tests=%d com/blank/attr=%d\n",tot,code,doc,tests,c+b+a}'
```

Scale the section set to the story: a small story may drop the diagrams and findings panel; a large one keeps all of them. The sections are a menu, not a fixed template.

## Step 3 — Change-card anatomy

Each card = an explanation block, then the code.

- **Kicker + title** — change kind (added / modified / moved) and the method or type name (monospace).
- **Tags** — `add` / `mod` badges and the FR(s) the change serves.
- **Explanation block** (the "what & why"), in short labelled fields: **What** (what changed), **Why it matters / process it participates in** (where this code sits in the runtime flow), and, where useful, a compact bullet list of **implementation notes** (invariants, non-obvious constraints, forward-tags).
- **Scrutiny callout** on risky units only — a coloured box (warn `▲` / risk `⚠` / info `i`) naming exactly what the reviewer should verify (a subtle invariant, a panic path, a security-relevant assumption). Do not put one on every card; reserve them for the genuinely load-bearing changes.
- **Code block** — the whole method, git-diff-styled: `+` lines green, `-` lines red, context neutral, with a small file/line caption. Trim long doc-comments (mark the elision) since the explanation block already carries the "why"; always show the actual changed lines in full.
- **Tooltips** — attach a `data-tip` to the meaningful changed lines (signature, the gate/guard line, the return, notable enum variants) explaining what that line does and why it came in or out. Not every line — the load-bearing ones.

## Step 4 — Visual design system

Utilitarian "IDE / code-review tool" treatment: information design first, polished but not flashy. Theme-aware (light + dark). Follow the `artifact-design` skill's fundamentals; the tokens below are the calibrated defaults for this page type.

**Colour** — cool-slate neutrals with a considered indigo accent; tuned (not garish) diff greens/reds; separate semantic colours for review severity.

- Light: bg `#eaeef4`, surface `#ffffff`, surface-2 `#f4f6fa`, ink `#1a2130`, muted `#5b6676`, border `#d9dfe9`, accent `#4a5fd6`, accent-soft `#e7eafb`; diff add bg `#e4f5ea` / edge `#1f9350`, del bg `#fdeaea` / edge `#d64545`; severity warn `#b7791f`, risk `#c23b3b`, ok `#1f9350`.
- Dark: bg `#0d1117`, surface `#151b24`, surface-2 `#1b222d`, ink `#e7ecf4`, muted `#98a4b6`, border `#27303c`, accent `#8496ff`, accent-soft `#1e2740`; diff add bg `#12261a` / edge `#2ea043`, del bg `#2a1618` / edge `#d05353`; warn `#e0b458`, risk `#f28d8d`, ok `#4ac47a`.

Define every colour as a CSS custom property on `:root`; redefine the tokens under `@media (prefers-color-scheme: dark)` and again under `:root[data-theme="dark"]` / `:root[data-theme="light"]` so the viewer's theme toggle wins in both directions. Style components only through the tokens.

**Type** — no webfonts (the Artifact CSP blocks font CDNs and a `@font-face` data URI is heavy). Use system stacks — this is the honest "IDE" vernacular and avoids silent fallback: UI `system-ui, -apple-system, "Segoe UI", Roboto, sans-serif`; code `ui-monospace, "SF Mono", Menlo, Consolas, monospace`. Uppercase eyebrows/kickers get letter-spacing; keep a clear type scale.

**Reusable markup + CSS patterns** (the parts that must match to look right):

Diff line — one element per line, sign via `::before`, colour via a left border + tinted background:

```
.cl{font-family:var(--mono);font-size:12.5px;line-height:1.62;white-space:pre;
    padding:0 16px 0 34px;position:relative;border-left:3px solid transparent}
.cl::before{position:absolute;left:12px;color:var(--faint)}
.cl.ctx::before{content:" "}
.cl.add{background:var(--add-bg);border-left-color:var(--add-edge)}
.cl.add::before{content:"+";color:var(--add-ink);font-weight:700}
.cl.del{background:var(--del-bg);border-left-color:var(--del-edge)}
.cl.del::before{content:"\2212";color:var(--del-ink);font-weight:700}
.cl.el{color:var(--faint);font-style:italic}.cl.el::before{content:"\22EF"}   /* elided doc-comment marker */
.cl[data-tip]{cursor:help}
.cl[data-tip]::after{content:"\25CF";position:absolute;right:12px;color:var(--accent);font-size:8px;opacity:.6} /* the hover-hint dot */
```

Each code line is then `<div class="cl add" data-tip="why this line came in">…HTML-escaped code…</div>`. Wrap every code block in a container with `overflow-x:auto` so the page body never scrolls sideways. HTML-escape `<`, `>`, `&` in code.

Cards, tables, chips: `surface` panels with `1px` `border` and a soft shadow; sticky table headers; `font-variant-numeric:tabular-nums` on any digit columns; pills for state (severity, gate/deferred) coloured from the semantic tokens, kept separate from the indigo accent.

## Step 5 — Interactions & diagrams

Keep JS tiny and inline (self-contained; the CSP blocks external scripts).

- **Scrollspy** — an `IntersectionObserver` over the sections that toggles an `active` class on the matching sidebar link.
- **Tooltip** — one fixed `#tip` div; on `mouseover` of any `[data-tip]`, set its text and position it near the cursor (flip when it would overflow the viewport); hide on `mouseout`. Respect `prefers-reduced-motion`.
- **Checklist** — toggle a strike-through class on the label when its checkbox changes (session-local; do not claim persistence).

**Diagrams** — Artifacts render mermaid natively from `<pre class="mermaid">…</pre>` (no library needed). Use them where the story has a real shape to show:

- a `stateDiagram-v2` for a lifecycle/state-machine story (states, legal edges, and any init-vs-transition distinction);
- a `flowchart TD` for a decision/gating story (the shared decision and its branches).

Mermaid label hygiene: quote labels containing parentheses (`B{"is_ready()?"}`), and avoid a raw `#` in a label (write `node-0`, not `node #0`) — it can break the parser.

## Step 6 — Publish

Write the HTML to a scratch file, set a `<title>`, then publish with the `Artifact` tool (self-contained: inline all CSS/JS, no external assets). Give it a stable favicon (e.g. `🔬`) and a one-sentence description. Re-publishing the **same file path** keeps the same URL, so iterate in place as the review turns up refinements. Artifacts are private by default; the reviewer shares from the page if they want.

## Quality rules

- **Accuracy over polish** — every shown line must come from the real diff/source; if a doc-comment is elided for length, mark it, and never elide a changed line.
- **Match density to the story** — a mechanical change needs fewer aids than a subsystem-spanning one; do not manufacture diagrams or findings that encode nothing.
- **One scrutiny callout per genuinely load-bearing change**, no more — over-flagging hides the real risks.
- **Self-contained** — no CDN scripts, fonts, or images; embed everything; both themes legible.

## Related Documents

- [`../AGENTS.md`](../AGENTS.md) — knowledge-base governance (English-only, compactness, index maintenance) that also governs this skills area.
- [`../moonblokz-index.md`](../moonblokz-index.md) — knowledge-base entry point; the Process Skills section links here.
- [`../moonblokz-blockchain-prd.md`](../moonblokz-blockchain-prd.md) and [`../moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) — the authoritative sources a blockchain-story review cites by FR number and architecture section.
