# ADR-0001: Two-repo meta layer — account-wide `.github` and family-scoped `crusty-meta`

- **Status:** Accepted
- **Date:** 2026-08-24
- **Deciders:** Amir Masri
- **Tracking issue:** none (2026-08-24 cross-repo consistency audit of crustywad / crustyview / crustygen)

## Context and problem statement

The three crusty repositories drifted apart: agent instruction files lived in different locations
per repo, a verbatim-copied `SECURITY.md` described another repository's security posture,
Conventional Commit enforcement ranged from a required CI check to nothing, and `.gitignore`
coverage was uneven. Shared conventions had no authoritative home, so every copy was free to rot
independently. Where should shared material live, and how should the repos stay in sync with it?

## Decision drivers

- Drift must be **loud**: a diverging copy should fail CI, not wait to be noticed.
- GitHub's default community health files apply to **every** repository under the account
  (30+ repos, most dormant) — anything published as a live default must be wanted everywhere.
- Some material is generic to how all repos are run; some is specific to the crusty family
  (inter-repo pinning rules, board policy, domain conventions).
- Cross-repo decisions need an ADR home; per-repo `docs/adr/` directories cannot hold them.

## Considered options

1. A single `masriamir/.github` repository for everything, generic and crusty-specific alike.
2. No meta repository — per-repo copies with a drift check only.
3. Two repositories: account-generic `masriamir/.github` plus family-scoped `crusty-meta`.

## Decision outcome

Chosen option: "two repositories", because the boundary test — *"would I want this on a dormant
2014 repository?"* — cleanly splits the material, and option 1 would either pollute the account-wide
defaults with crusty specifics or dilute them to uselessness.

- **`masriamir/.github`** (account-generic): live defaults limited to `CODE_OF_CONDUCT.md` and a
  generic `SECURITY.md` (issue/PR templates are deliberately **not** live defaults); canonical
  `templates/` and `scripts/` consumed by sync; reusable workflows (`pr-title`, `meta-check`);
  Claude Code process skills.
- **`crusty-meta`** (family-scoped): the family README (roles, inter-repo pinning rules, release
  and board policy), cross-repo ADRs, and crusty-specific template blocks.
- **Sync mechanism:** each repo carries a `.meta-manifest` whose entries pin
  `source repo @ SHA : path → destination` (whole-file or marker-delimited block); a local
  `meta-check`/`meta-sync` recipe and a reusable CI job diff against the pin, so upstream changes
  never surprise-redden unrelated PRs and adopting them is an explicit pin bump.

### Consequences

- Good, because every shared file has exactly one authoritative copy and divergence fails CI.
- Good, because the community-health defaults reach all repositories without touching them.
- Bad, because two more repositories carry their own CI, rulesets, and review loops.
- Neutral, because pinned-SHA sync makes updates explicit per repo rather than automatic.

## More information

Companion decision (same audit, verified against the Claude Code and GitHub Copilot documentation
on 2026-08-24): each repo's instruction files converge on a root `AGENTS.md` (single tool-neutral
source; Copilot code review reads root `AGENTS.md` and `.github/copilot-instructions.md`, not
`CLAUDE.md`), a root `CLAUDE.md` that imports it via `@AGENTS.md` plus Claude-only notes, a short
reviewer-focused `.github/copilot-instructions.md`, and path-scoped `.claude/rules/*.md`.
The three repos' `Main Branch` rulesets were aligned the same day (approval count, required
checks, `code_scanning`/`update` rules). Revisit this ADR if the family gains repos with
materially different tooling, or if the sync manifest proves too heavy for its value.
