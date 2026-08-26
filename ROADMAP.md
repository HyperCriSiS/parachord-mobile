# Parachord Mobile Fork Roadmap

This roadmap tracks the work planned for this fork while keeping upstream compatibility as a primary constraint.

The long-term goal is to extend Parachord into a highly configurable multi-service music client without replacing the architecture that already works well upstream.

## Guiding principles

- Keep changes modular and upstream-friendly wherever practical.
- Prefer generic capabilities over provider-specific branches in shared core code.
- Preserve native provider features instead of flattening every service into a lowest-common-denominator API.
- Treat playback source, metadata source, recommendation source, and sync source as separate concerns.
- Keep provider-specific protocol complexity behind narrow adapters.
- Reuse upstream architecture and conventions before introducing parallel abstractions.
- Add tests before or together with non-trivial behavior changes.
- Maintain Android/iOS/KMP parity where the upstream architecture expects it.
- Keep AI-dependent functionality optional. Core recommendation behavior must remain deterministic and useful without AI.
- Keep the fork easy to rebase onto upstream Parachord.

## Phase 0 — Baseline and upstream alignment

- [ ] Record current upstream commit used by the fork.
- [ ] Audit Android, iOS and shared module build status.
- [ ] Audit existing CI workflows before enabling additional fork-only jobs.
- [ ] Establish an upstream sync/rebase workflow.
- [ ] Identify fork-only configuration that should never be proposed upstream.
- [ ] Keep documentation of architectural decisions in `docs/`.
- [ ] Review current `CLAUDE.md`, `AGENTS.md` and upstream implementation rules before each architectural change.

## Phase 1 — Recommendation architecture foundation

Detailed design: [`docs/RECOMMENDATION_SYSTEM.md`](docs/RECOMMENDATION_SYSTEM.md)

- [ ] Inventory every existing Parachord recommendation/discovery surface.
- [ ] Build a capability matrix for external services before implementing new integrations.
- [ ] Introduce a provider-agnostic `RecommendationSurface` model.
- [ ] Introduce a generic `RecommendationSource` contract.
- [ ] Preserve every native recommendation surface as independently usable.
- [ ] Migrate the current Last.fm + ListenBrainz recommendation implementation to the generic contract without changing user-visible behavior first.
- [ ] Add explicit surface categories such as discovery, replay, radio, new releases and editorial.
- [ ] Add merge eligibility / merge-pool metadata instead of assuming all recommendations are comparable.
- [ ] Generalize the current Last.fm + ListenBrainz merge into a reusable merger.
- [ ] Add source/surface weighting.
- [ ] Add filtering and sorting controls.
- [ ] Add configurable presets such as For You, Discovery, New Releases and Track Radio.
- [ ] Add expert-mode custom merge pools.
- [ ] Design optional `.axe` recommendation-source capability.
- [ ] Keep AI recommendations as an optional source, not a dependency of the core engine.

## Phase 2 — Recommendation capability research

Create a verified matrix for each service, including authentication requirements, API stability, personalization level, feedback support, identifiers and whether the surface is suitable for merging.

Initial targets:

- [ ] ListenBrainz
  - raw collaborative-filter recommendations and scores
  - Daily Jams
  - Weekly Jams
  - Weekly Exploration
  - recommendation feedback
- [ ] Last.fm
  - recommended radio/station
  - similar tracks
  - similar artists
  - tags and scrobble-derived taste signals
- [ ] Deezer
  - Flow
  - track mixes
  - artist mixes
  - personalized Home
  - Made for Me / SmartTracklists where available
  - new releases / editorial discovery
- [ ] YouTube Music
  - personalized Home shelves
  - Quick Picks
  - Listen Again
  - radio/watch queue
  - related tracks
  - mood/genre discovery
  - feedback tokens where available
- [ ] Apple Music
  - personal recommendations
  - recommendation reasons
  - stations
  - heavy rotation
  - recently played
  - new releases / editorial surfaces
- [ ] Spotify
  - current user taste signals and history
  - currently accessible recommendation surfaces under present API restrictions
- [ ] SoundCloud
  - related tracks
  - related artists/users
  - discovery surfaces
- [ ] TIDAL
  - personalized mixes
  - discovery/new-release surfaces
  - API/SDK availability and restrictions
- [ ] Qobuz
  - similar artists
  - album suggestions
  - track recommendations
  - discover/editorial surfaces
- [ ] Bandcamp
  - discovery/fan/collection signals that can be exposed meaningfully
- [ ] Local library
  - play history
  - favorites
  - skips
  - repeats
  - playlist membership
  - optional acoustic/metadata similarity later

For every surface, document:

- source/provider
- surface ID
- semantics
- personalized vs editorial
- seed requirements
- item types returned
- native score/rank availability
- feedback capability
- stable identifiers (ISRC, MBID, provider IDs)
- authentication requirements
- rate limits
- merge compatibility
- API/legal caveats

## Phase 3 — Deezer / OpenDeezer integration

Use OpenDeezer as the Deezer protocol/backend implementation rather than duplicating Deezer reverse-engineering inside Parachord.

Target architecture:

```text
Parachord
  ├─ Deezer resolver
  ├─ Deezer sync provider
  ├─ Deezer recommendation surfaces
  ├─ Deezer metadata/library adapter
  └─ Deezer playback adapter
         ↓
     narrow bridge
         ↓
     OpenDeezer Go core
```

- [ ] Define a stable bridge API/version handshake.
- [ ] Keep OpenDeezer independently updateable.
- [ ] Implement Deezer authentication.
- [ ] Implement catalog/search mapping into Parachord models.
- [ ] Implement favorites/library mapping.
- [ ] Implement playlist sync capabilities where safe.
- [ ] Implement Deezer recommendation surfaces individually.
- [ ] Implement full-track playback while keeping Parachord's unified queue/player architecture.
- [ ] Evaluate Media3 custom DataSource vs loopback decrypted stream for Android.
- [ ] Define iOS strategy separately if OpenDeezer support is not directly portable.
- [ ] Add graceful fallback to other resolvers if Deezer playback is unavailable.

## Phase 4 — YouTube Music mobile integration

Parachord Mobile currently lacks the complete native YouTube Music path desired by this fork.

- [ ] Study current FuoEvolve and NeriPlayer approaches as references without copying incompatible code.
- [ ] Design a native YT Music resolver/provider path.
- [ ] Implement robust authentication/session handling.
- [ ] Separate anonymous/catalog behavior from authenticated personalized behavior.
- [ ] Implement playback URL resolution with refresh/recovery behavior.
- [ ] Integrate YT Music recommendation surfaces individually.
- [ ] Add YT Music surfaces to appropriate unified recommendation pools.
- [ ] Preserve provider fallback through the existing resolver architecture.

## Phase 5 — Recommendation ranking and personalization

- [ ] Normalize provider-native scores without pretending unlike scores are directly comparable.
- [ ] Support source weight and surface weight independently.
- [ ] Make consensus an optional weak signal, never an automatic winner.
- [ ] Add novelty controls.
- [ ] Add familiarity controls.
- [ ] Add artist repetition/diversity controls.
- [ ] Add genre/mood filters where metadata permits.
- [ ] Add popularity filtering/weighting where data exists.
- [ ] Add release-age filters.
- [ ] Add filters for live/remix/cover/instrumental variants where reliably identifiable.
- [ ] Add explicit artist/track/label blocking.
- [ ] Add already-heard/favorite/playlist-membership filters.
- [ ] Track local recommendation outcomes such as skip, completion, replay, favorite and playlist-add.
- [ ] Evaluate adaptive per-source and per-context weights.
- [ ] Keep learned ranking inspectable and resettable by the user.

## Phase 6 — Recommendation plugin extensibility

- [ ] Extend `.axe` manifest capabilities with recommendation support.
- [ ] Define plugin-side surface discovery.
- [ ] Define plugin-side candidate schema.
- [ ] Define optional feedback callbacks.
- [ ] Add timeouts, cancellation and rate-limit behavior.
- [ ] Add provenance/trust information for third-party recommendation plugins.
- [ ] Add manual `.axe` installation from file.
- [ ] Add installation from URL.
- [ ] Consider additional plugin repositories besides the official marketplace.
- [ ] Add integrity/signature/hash support where practical.
- [ ] Expose installed source permissions/capabilities clearly to the user.

## Phase 7 — Cross-provider identity and resolution

- [ ] Prefer ISRC where reliable.
- [ ] Use MusicBrainz recording/release identifiers as additional identity anchors.
- [ ] Preserve provider IDs for direct playback fast paths.
- [ ] Improve fuzzy identity fallback while avoiding remaster/live/cover false matches.
- [ ] Reuse Parachord's resolver confidence model for recommendation candidates.
- [ ] Make recommendation origin independent from playback source.
- [ ] Allow the user to choose playback provider priority independently from recommendation source priority.

Example:

```text
Recommended by Deezer Flow
        ↓
canonical track identity
        ↓
resolver pipeline
        ↓
Local / Deezer / Apple Music / Spotify / SoundCloud / ...
```

## Phase 8 — Discovery UX

- [ ] Keep provider-native recommendation surfaces directly accessible.
- [ ] Add a Unified recommendations entry alongside provider-native entries.
- [ ] Group recommendation surfaces by provider and semantic type.
- [ ] Make recommendation origin visible on demand.
- [ ] Expose "Why am I seeing this?" where source data allows it.
- [ ] Add per-surface enable/disable controls.
- [ ] Add simple presets for normal users.
- [ ] Add expert controls without making the default UI complicated.
- [ ] Allow saving custom recommendation profiles/mix pools.

## Phase 9 — Additional service expansion

After the recommendation/provider contracts are stable, evaluate integrations in this order based on technical value and legal/API feasibility:

- [ ] TIDAL
- [ ] Qobuz
- [ ] additional SoundCloud capabilities
- [ ] additional Apple Music capabilities
- [ ] additional Spotify capabilities permitted by current API policy
- [ ] Bandcamp enhancements
- [ ] self-hosted/local providers such as Navidrome/Subsonic where useful

## Phase 10 — i18n and localization

Parachord currently contains many hard-coded English strings.

- [ ] Audit Android, iOS and shared user-visible strings.
- [ ] Establish a maintainable localization architecture.
- [ ] Preserve upstream compatibility.
- [ ] Add German as the first full additional language.
- [ ] Add completeness checks to CI.
- [ ] Ensure plugin-provided labels support localization/fallbacks.

## Phase 11 — Quality, security and maintainability

- [ ] Maintain unit tests for pure merge/ranking logic.
- [ ] Add fixture-based cross-provider recommendation tests.
- [ ] Add deterministic ranking tests.
- [ ] Add malformed/untrusted plugin payload tests.
- [ ] Add cancellation and partial-source-failure tests.
- [ ] Ensure one failed recommendation source cannot suppress other sources.
- [ ] Add cache/stale-while-revalidate behavior per surface where appropriate.
- [ ] Add diagnostics for source failures without exposing credentials.
- [ ] Keep secrets in secure storage.
- [ ] Review third-party provider terms before shipping integrations.
- [ ] Avoid unnecessary GitHub Actions runs in the fork.

## Upstream strategy

Prefer contributing generic improvements upstream when they stand on their own without fork-specific services.

Good upstream candidates:

- generic recommendation-surface model
- generic recommendation-source contract
- general merge/filter/ranking infrastructure
- `.axe` recommendation capability
- plugin import UX if broadly useful
- tests and cross-platform abstractions

Likely fork-specific initially:

- OpenDeezer bridge
- reverse-engineered Deezer details
- experimental provider integrations
- aggressive expert ranking controls until UX is proven
- experimental adaptive/learned ranking

The fork should not depend on upstream accepting any proposal; however, keeping generic work upstreamable minimizes long-term divergence.
