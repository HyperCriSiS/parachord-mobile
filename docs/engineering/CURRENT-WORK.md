# Current Work — parachord-mobile

Last reviewed: 2026-10-08. The canonical project-wide state is the default `main` branch. This document is a compact **handoff**, not a duplicate roadmap or proof of completion.

## Source of truth and live evidence

- [Root roadmap](../../ROADMAP.md) — milestones and pending work.
- [Open pull requests](https://github.com/HyperCriSiS/parachord-mobile/pulls) and [issues](https://github.com/HyperCriSiS/parachord-mobile/issues) — live review queue; verify before working.
- [GitHub Actions](https://github.com/HyperCriSiS/parachord-mobile/actions) — applicable build/test/validation results; no claim of physical hardware testing unless documented.
- [Documentation index](../DOCUMENTATION.md) and [agent instructions](../../AGENTS.md) — further project rules.

## Next-work orientation

The fork-specific `ROADMAP.md` describes planned cross-service/KMP work. Check current upstream changes, Android/mobile build evidence, and `docs/plans/` before deciding whether a planned item is implemented. Keep the existing `AGENTS.md` → `CLAUDE.md` instruction relationship intact.

## Handoff discipline

Check the latest integrated `main` and open branches; select a bounded task with explicit acceptance tests. Update the roadmap after verified integration. Record a blocker or next step here with a PR/issue/test link, not an unverified status or stale commit identifier. Do not overwrite canonical progress from an older feature branch.
