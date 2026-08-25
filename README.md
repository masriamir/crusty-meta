# crusty-meta

How the **crusty** family of repositories is run: shared conventions, cross-repo architecture
decisions, and canonical templates.

Account-generic conventions (branching, commits, PR titles, review loop, action pinning) live in
[`masriamir/.github`](https://github.com/masriamir/.github); this repository holds only what is
specific to the crusty family.

## The family

| Repo | Role |
|---|---|
| [crustywad](https://github.com/masriamir/crustywad) | Core library + `cwad` CLI: performant, safe, typed Doom WAD I/O in Rust; published to crates.io |
| [crustyview](https://github.com/masriamir/crustyview) | Web-based WAD reader/viewer: Rust→WASM core behind a Svelte + TypeScript shell |
| [crustygen](https://github.com/masriamir/crustygen) | Compiles a room-graph IR into UDMF `TEXTMAP` and playable Doom PWADs |

## Inter-repo rules

- Downstream repos (`crustyview`, `crustygen`) depend on **pinned crustywad releases** from
  crates.io — never a path or git dependency. The escape hatch for urgent fixes is a commented
  `[patch.crates-io]` override in the downstream `Cargo.toml`.
- When a crustywad pin bumps, the downstream repo mirrors crustywad's `rust-version` (MSRV);
  crustyview's CI fails on drift.
- API friction or a bug found downstream → file an **issue on crustywad**, fix it on crustywad's
  `main`, bump the downstream pin on the next release.

## Releases

- **crustywad** — automated by `release-plz` from Conventional Commits on `main`: merging the
  release PR publishes both crates to crates.io via Trusted Publishing and pushes tags; `dist`
  builds the cross-platform `cwad` binaries from the `crustywad-cli-v*` tag.
- **crustyview** — cut locally with `just release` (git-cliff), one product version, tag
  `v<version>`; not published to crates.io.
- **crustygen** — no release automation yet; built from source.

## Project tracking

- `crustywad` and `crustyview` share GitHub **Project #5 "Crustywad"** — issue-per-change, with
  `Status` (`Backlog → Ready → In progress → In review → Done`) and `Horizon` (`Now/Next/Later`)
  fields; epics use native sub-issues.
- **Milestones are release-scoped and scope-named** (never version numbers — versions are derived
  by tooling at ship time); shipped versions are recorded in the milestone description at closeout.
- Per-repo workflow details (gates, board transitions, review loop) live in each repository's own
  agent/contributor docs.

## Layout

- [`docs/adr/`](docs/adr/) — **cross-repo** architecture decision records (decisions that span the
  family; single-repo decisions stay in that repo's own `docs/adr/`)
- `templates/` (planned) — crusty-specific canonical blocks consumed by each repo's
  `.meta-manifest.toml` via the shared `meta-check` drift check

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or
[MIT license](LICENSE-MIT) at your option.
