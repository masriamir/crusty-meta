# ADR-0003: Account-wide Codecov policy adoption

- **Status:** Accepted
- **Date:** 2026-08-30
- **Deciders:** Amir Masri
- **Tracking issue:** [#4](https://github.com/masriamir/crusty-meta/issues/4)

## Context and problem statement

Every Crusty repository gates coverage through Codecov, and each had arrived at its own
`codecov.yml` independently. The three files agreed on nothing except that they existed:
crustywad set only a patch status, crustyview set `project: auto`/1% with `patch: 80%`,
crustygen set `project: auto`/1% with `patch: 90%`. Two of the three additionally carried
`require_changes: true` and a `layout` naming `reach`.

Those last two are not stylistic drift. `require_changes: true` suppresses the Codecov PR
comment on any pull request that does not move coverage, and every repository here syncs the
[`copilot-review-loop`](https://github.com/masriamir/.github/blob/main/templates/blocks/copilot-review-loop.md)
block, which makes "the codecov comment reports no uncovered changed lines" a precondition for
human review. On a docs-only, CI-config, or pure-refactor PR the gate was therefore
**unsatisfiable**: it waited on a comment Codecov would never post. `reach` is no longer a layout
component Codecov documents, so it rendered nothing.

Which parts of a coverage policy are genuinely family-wide, where does the one canonical copy
live, and how do repositories adopt it without losing the per-repo reasoning their current files
encode?

## Decision drivers

- The same drift mechanics ADR-0002 solved for instruction text apply to coverage config: one
  authoritative copy, and a diverging copy must redden `meta-check` rather than rot quietly.
- The canonical policy must stay **account-generic**. It is owned by `masriamir/.github` and is
  offered to every repository on the account, not just this family; nothing Crusty-specific may
  leak into it.
- Per-repo reasoning is real and must survive. crustyview's wasm scope note and crustygen's
  `liftprobe` exclusion are load-bearing explanations, not clutter.
- Adoption must be opt-in per fragment, so a repository can take the comment policy without the
  status targets, or vice versa.
- A coverage gate that silently does not run is worse than no gate — the same class of problem as
  the `continue-on-error` removed in masriamir/crustyview#69.

## Considered options

1. Leave each repository's `codecov.yml` fully local; fix the `require_changes` bug in place,
   three times.
2. Sync one whole `codecov.yml` file into every repository.
3. Extract the family-wide parts to canonical **marker-delimited block fragments** in
   `masriamir/.github`, block-synced into each repository's own `codecov.yml`, with everything
   else staying local outside the markers.

## Decision outcome

Chosen option: **option 3**, reusing the ADR-0002 block machinery unchanged.

Option 1 is the status quo that produced three divergent files and let the same bug sit in two of
them. Option 2 fails on the per-repo content: `ignore` lists, scope comments, and exclusion
rationales differ materially between a WAD library, a WASM viewer, and a map generator, and a
single file would either drop them or force every repository to carry the others'.

Two canonical fragments, three adoptions:

| Fragment | Marker | Destination |
|---|---|---|
| `codecov-status-default.yml` | `codecov-project-status` | `coverage.status.project` |
| `codecov-status-default.yml` | `codecov-patch-status` | `coverage.status.patch` |
| `codecov-comment.yml` | `codecov-comment` | `comment` |

The status fragment is **one file adopted twice**. Project and patch share a target, so they share
a source and cannot be edited into disagreement.

### Consumer set and exceptions

All four consumers pin the same upstream commit —
`8078029680e0e50db226cf506586bb4bdcbb545f`, the merge of
[masriamir/.github#13](https://github.com/masriamir/.github/pull/13):

| Repository | Pin | Status fragments | Comment fragment | Local content preserved |
|---|---|---|---|---|
| crustywad | `8078029` | yes — patch already 90%, project newly gated | yes | patch overflow-guard note |
| crustyview | `8078029` | yes — patch 80% → 90%, project `auto` → 90% | yes | full wasm SCOPE comment; no `ignore` added |
| crustygen | `8078029` | yes — project `auto`/1% → 90% | yes | `ignore: examples/**` and `liftprobe` rationale |
| crustyllm | `8078029` | **no** | **no** | — |

`crustyllm` takes the shared-file pin only. It is out of scope for the coverage policy under this
ADR; revisit when its coverage story is settled.

Adopting a status fragment requires that the repository **already uploads coverage**, because the
fragment sets `if_not_found: failure`, inverting Codecov's default under which a status with no
report silently passes. Adopting before uploads work leaves both statuses red.

### Staged, upstream-first rollout

The order is not incidental — each stage is a precondition for the next.

The sync tool is `masriamir/.github`'s `scripts/meta_sync.py`. It does not live in this
repository; each consumer vendors a copy to the same path, and that vendored copy is itself
manifest-tracked, so the tool is both the thing doing the syncing and a thing being synced. That
is what makes the order below load-bearing.

1. **Canonical first.** The fragments are authored and merged in `masriamir/.github`
   ([#13](https://github.com/masriamir/.github/pull/13)). Nothing is adopted against an unmerged
   ref: `masriamir/.github`'s `scripts/meta_sync.py` requires a 40-character commit SHA, and only
   a merged commit is stable.
2. **Tool before content.** A consumer's vendored `scripts/meta_sync.py` is itself
   manifest-tracked, and the version predating masriamir/.github#13 splices canonical bytes
   verbatim. The Codecov fragments are nesting-neutral and rely on the re-indenting sync
   introduced there, so each consumer must bump and sync the **vendored tool** before adding the
   Codecov entries. The other order writes un-indented bytes into `coverage.status.*` and produces
   invalid YAML that the byte-oriented drift check cannot see.
3. **Per-repo adoption.** One issue and one pull request per repository, each pinned to the same
   merged upstream commit.
4. **Verification.** Below.

### Codecov project-status verification requirement

**A project status that never posts is indistinguishable from one that passes.** Adoption is not
complete when `meta-check` is green; it is complete when the status is *observed* on a real PR.

crustyview [#99](https://github.com/masriamir/crustyview/issues/99) documented `codecov/project`
never posting across eight commits while `codecov/patch` posted reliably from the same block, and
proposed `target: auto` failing to resolve a base as a candidate cause. The rollout tested that
incidentally, since the shared fragment uses an absolute target and needs no base.

It did not hold. On the three adoption PRs — with absolute `target: 90%` — `codecov/patch` posts
and **`codecov/project` still does not, in all three repositories**. So the behavior is not
crustyview-specific and not caused by `target: auto`; that eliminates masriamir/crustyview#99's
third candidate and points at a Codecov-side cause (an account or repository setting overriding
the status, or a plan restriction).

Therefore, for any repository adopting the status fragments:

- Check both GitHub surfaces, since Codecov appears as a check run on PR heads and as a commit
  status on the default branch:
  ```bash
  gh api repos/OWNER/REPO/commits/<sha>/check-runs
  gh api repos/OWNER/REPO/commits/<sha>/status
  ```
- Treat a missing `codecov/project` as an **open defect**, not an adoption failure. The config is
  correct — it validates against `https://codecov.io/validate` — so the fault is downstream.
- Do not require `codecov/project` in a ruleset until it has been observed posting. Requiring a
  status that never arrives blocks every merge.

### Consequences

- Good, because the coverage target has one source; a drifted copy reddens `meta-check`.
- Good, because the `require_changes` defect is fixed once, upstream, rather than three times.
- Good, because per-repo reasoning survives verbatim outside the markers.
- Bad, because raising crustyview's patch target 80% → 90% tightens a gate against a documented
  rationale about wasm code being structurally invisible to `cargo llvm-cov`. All three
  repositories sit at 98–99%, so there is roughly eight points of headroom — but the reasoning in
  that scope comment matters more after the change, not less.
- Bad, because adoption is a coordinated per-repo pin bump; a future policy edit repeats it.
- Neutral, because the fragments remain account-generic and opt-in. This ADR records that the
  Crusty family adopts them; it does not make them Crusty-owned.

## Pros and cons of the options

### Option 1 — fully local `codecov.yml` per repository

- Good, because zero machinery and total per-repo freedom.
- Bad, because it produced three divergent files and left the same gate-breaking
  `require_changes: true` sitting in two of them.

### Option 2 — one whole-file `codecov.yml` synced everywhere

- Good, because trivially consistent.
- Bad, because `ignore` lists and scope rationales differ materially per repository; the file
  would either discard them or impose each repository's specifics on the others.

## More information

- Canonical fragments and their adoption guide:
  [`templates/blocks/README.md`](https://github.com/masriamir/.github/blob/main/templates/blocks/README.md).
- The policy is owned by `masriamir/.github` per [ADR-0001](0001-two-meta-repo-architecture.md);
  the block-sync machinery is [ADR-0002](0002-instruction-file-consolidation.md).
- Revisit when: `codecov/project` is diagnosed (crustyview#99), `crustyllm`'s coverage story
  settles, or a repository needs a target the shared fragment cannot express — in which case the
  documented opt-out is to drop the manifest entry **and delete the marker lines and their body**,
  since an orphaned marker keeps its last-synced content.
