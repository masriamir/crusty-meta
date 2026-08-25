# ADR-0002: Instruction-file consolidation and shared-block centralization

- **Status:** Accepted
- **Date:** 2026-08-24
- **Deciders:** Amir Masri
- **Tracking issue:** none (2026-08-24 cross-repo consistency audit; Crusty TODO "General" items 2, 3, 8)

## Context and problem statement

[ADR-0001](0001-two-meta-repo-architecture.md) established the meta layer and sketched an
instruction-file convergence (a tool-neutral root `AGENTS.md`, a `CLAUDE.md` that imports it, a
short `.github/copilot-instructions.md`) but left it unbuilt. On the ground the three repos had
drifted: crustywad kept its guidance in `.claude/CLAUDE.md`, crustyview in a root `CLAUDE.md`,
and crustygen had none at all; and each repo had independently re-derived the *same* conventions
in *different words* — crustywad and crustyview each grew their own American-English section (with
the same hard-won "state the pattern, skip backticked spans" insight, worded differently), their
own Conventional-Commit/PR-title guidance, and their own board-transition table. Which conventions
are genuinely shared, where should the one canonical copy live, and how do the repos consume it
without re-introducing drift?

## Decision drivers

- One authoritative copy per shared convention; a diverging copy must fail CI, not rot quietly.
- Instruction files must stay **tool-neutral where shared** (Copilot code review reads root
  `AGENTS.md` and `.github/copilot-instructions.md`, never `CLAUDE.md`; Claude reads `CLAUDE.md`
  and only sees `AGENTS.md` through an explicit `@AGENTS.md` import).
- Repo-specific guidance (build/test commands, stack conventions, CI, feature flags) must stay
  with its repo — over-centralizing it would couple unlike repos.
- Concise files matter: excessive or duplicated instruction text degrades LLM performance
  (General item 8), so consolidation is also a trimming pass.
- Reuse the sync machinery already built (ADR-0001 `.meta-manifest.toml` block mode), not a new one.

## Considered options

1. Keep per-repo instruction files; rely on a periodic manual content audit for consistency.
2. Sync a single whole-file `AGENTS.md` into every repo.
3. Repo-authored `AGENTS.md` per repo, with the genuinely-common conventions extracted to
   canonical **marker-delimited blocks** in `masriamir/.github`, block-synced into each
   `AGENTS.md` and drift-checked by `meta-check`.

## Decision outcome

Chosen option: **option 3**. Option 1 is the status quo that produced the drift; option 2 is wrong
because the repos' non-shared guidance genuinely differs (a Rust WAD library, a WASM viewer, a map
generator) and a single file would either omit it or force-fit it.

**Canonical shared blocks** live in `masriamir/.github` under `templates/blocks/` — one file per
block, each the single authored version of a convention that is identical in intent across the
family:

| Block | Covers |
|---|---|
| `language-en-us` | American-English spelling everywhere + third-party-vocabulary exception + the "state the pattern, skip backticked spans, not a mechanical find-and-replace" guidance |
| `commit-conventions` | Conventional Commits; the PR title **is** the squash commit / changelog entry / version bump; blank squash body; never `gh pr create --fill` |
| `branch-naming` | `<type>/<slug>` with `type ∈ {feature,bugfix,hotfix,docs,chore}`, issue number optional, descriptive slug required; branch from `main` |
| `board-transitions` | Agent-driven GitHub Project Status flow: Backlog → Ready → In progress → In review → Done (only Done is board-automated) |
| `copilot-review-loop` | Ready-for-human-review = every bot thread resolved **and** required CI green **and** codecov clean; use the review-resolution skill |

**Per-repo file layout** (all three repos converge on this):

- **Root `AGENTS.md`** — repo-authored sections (overview, layout, build/test, stack conventions,
  CI, feature flags) **plus** the shared blocks, each wrapped in
  `<!-- >>> meta:<block> -->` / `<!-- <<< meta:<block> -->` markers and kept current by a
  `mode = "block"` entry in the repo's `.meta-manifest.toml`.
- **Root `CLAUDE.md`** — begins with `@AGENTS.md`, then Claude-only material (memory workflow,
  skill usage, board mechanics). crustywad's `.claude/CLAUDE.md` moves to the repo root.
- **`.github/copilot-instructions.md`** — short and reviewer-focused; it does **not** duplicate the
  shared blocks (Copilot already reads `AGENTS.md`), it points at them.
- **crustygen** gains the whole set, authored from this template.

### Consequences

- Good, because each shared convention has exactly one source; a drifted copy reddens `meta-check`.
- Good, because both toolchains inherit the shared blocks (Copilot reads `AGENTS.md` directly;
  Claude via `@AGENTS.md`) without the text being duplicated per tool.
- Good, because the split forces a concision pass (General item 8) and gives crustygen parity.
- Bad, because the initial migration rewrites three repos' instruction files and authors five
  canonical blocks — a large one-time editorial effort.
- Neutral, because adopting a canonical-block change stays an explicit per-repo pin bump, as with
  every other synced file.

## Pros and cons of the options

### Option 1 — per-repo files + manual audit

- Good, because zero migration and maximum per-repo freedom.
- Bad, because it is exactly what produced the drift; consistency depends on someone remembering.

### Option 2 — one whole-file `AGENTS.md` synced everywhere

- Good, because trivially consistent.
- Bad, because the repos' non-shared guidance differs materially; the file would omit real
  per-repo needs or bloat every repo with the others' specifics.

## More information

Implements the companion decision recorded in [ADR-0001](0001-two-meta-repo-architecture.md)
"More information", using the block-sync mechanism from that ADR (whose byte-for-byte and
no-final-newline-block-convergence corrections landed in `masriamir/.github#5`). Rollout order:
canonical blocks in `masriamir/.github` → crustywad as the reference implementation → replicate to
crustyview and crustygen, one PR per repo. Revisit if a shared block needs to diverge for one repo
(promote it back to repo-authored) or if the family gains a repo whose conventions don't fit the
five blocks.
