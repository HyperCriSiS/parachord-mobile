# Recommendation System Architecture

Status: initial design / planning document

This document defines the intended recommendation architecture for the Parachord fork. It intentionally separates provider-native recommendation experiences from optional cross-provider recommendation merging.

## 1. Core idea

A music service does not have one generic "recommendations" endpoint. It normally exposes several recommendation experiences with different semantics.

Examples:

```text
Deezer
  ├─ Flow
  ├─ Track Mix
  ├─ Artist Mix
  ├─ Made for Me
  └─ personalized Home sections

YouTube Music
  ├─ Quick Picks
  ├─ Listen Again
  ├─ Radio
  ├─ Related
  └─ personalized Home shelves

ListenBrainz
  ├─ raw collaborative-filter recommendations
  ├─ Daily Jams
  ├─ Weekly Jams
  └─ Weekly Exploration
```

These individual experiences are called **Recommendation Surfaces** in this design.

A provider is the service. A surface is one concrete recommendation experience provided by that service or recommendation system.

Examples:

```text
provider = deezer
surface  = flow

provider = ytmusic
surface  = quick_picks

provider = listenbrainz
surface  = weekly_exploration
```

## 2. Two equally important modes

The system must support both modes from the beginning.

### 2.1 Standalone native surfaces

Every provider-native recommendation surface remains independently usable.

Examples:

- open Deezer Flow directly
- open YouTube Music Listen Again directly
- open ListenBrainz Weekly Exploration directly
- start a Deezer artist mix directly
- start a YouTube Music radio from one seed track directly

The unified engine must never replace or hide these experiences.

### 2.2 Unified / merged recommendations

Compatible surfaces may additionally feed one or more unified pools.

Examples:

```text
Unified Discovery
  ← Deezer discovery/personalized candidates
  ← YouTube Music Quick Picks
  ← ListenBrainz raw recommendations
  ← ListenBrainz Weekly Exploration
  ← Last.fm recommendations
```

or:

```text
Unified Track Radio
  ← Deezer Track Mix
  ← YouTube Music Radio / Related
  ← Last.fm Similar Tracks
  ← SoundCloud Related
```

The merge is therefore an extra layer, not a replacement for provider functionality.

## 3. Why surfaces matter

A single `getRecommendations(provider)` abstraction loses important semantics.

These are not equivalent:

- replaying already-known music
- discovering unknown music
- finding tracks related to a seed song
- finding music related to a seed artist
- personalized daily mixes
- new releases
- editorial picks
- charts
- mood/genre browsing

A merge engine should not automatically combine unlike recommendation types.

## 4. Proposed surface categories

Initial semantic categories:

```text
DISCOVERY
PERSONALIZED_FEED
PERSONALIZED_RADIO
TRACK_RADIO
ARTIST_RADIO
RELATED_TRACKS
RELATED_ARTISTS
REPLAY
NEW_RELEASES
EDITORIAL
CHARTS
MOOD
GENRE
DAILY_MIX
WEEKLY_MIX
AI_GENERATED
OTHER
```

This list should remain extensible. A surface may expose more than one semantic tag if needed.

## 5. Proposed merge policy

A surface should declare how it participates in unified pools.

Do not use only a boolean `mergeable` flag. Use semantic compatibility.

Example model:

```text
RecommendationSurface
  id
  providerId
  title
  description
  categories[]
  inputRequirements
  outputTypes[]
  personalization
  mergePools[]
  supportsFeedback
  supportsPagination
  supportsContinuousPlayback
```

Example:

```text
deezer.flow
  categories = [PERSONALIZED_RADIO, DISCOVERY]
  mergePools = [FOR_YOU, DISCOVERY]

ytmusic.listen_again
  categories = [REPLAY]
  mergePools = [REPLAY]

lastfm.similar_tracks
  categories = [RELATED_TRACKS, TRACK_RADIO]
  mergePools = [TRACK_RADIO]
```

Users in expert mode may override defaults and build custom pools.

## 6. Recommendation source contract

The native core should define a provider-agnostic contract. Exact Kotlin naming is intentionally left open until implementation work begins.

Conceptually:

```kotlin
interface RecommendationSource {
    val sourceId: String

    suspend fun surfaces(): List<RecommendationSurface>

    suspend fun load(
        request: RecommendationRequest
    ): RecommendationResult

    suspend fun submitFeedback(
        feedback: RecommendationFeedback
    ): FeedbackResult
}
```

A full music provider may implement this capability as one part of its provider integration.

Examples:

```text
Deezer provider
  ├─ auth
  ├─ search
  ├─ playback
  ├─ library/sync
  └─ recommendation source

YouTube Music provider
  ├─ auth
  ├─ search
  ├─ playback
  └─ recommendation source
```

Recommendation-only systems can implement the same contract without becoming playback providers:

```text
ListenBrainz recommendation source
Last.fm recommendation source
future community recommendation plugin
```

The engine should not care whether a source is native, part of a music provider, or supplied by a plugin.

## 7. Candidate model

Do not discard provider-native information during normalization.

Conceptual model:

```text
RecommendationCandidate
  identity
    title
    artist
    album
    isrc?
    recordingMbid?
    releaseMbid?
    providerIds{}

  origin
    sourceId
    surfaceId
    providerId?
    rank?
    nativeScore?
    reason?
    generatedAt?

  context
    personalized
    seedTrack?
    seedArtist?
    genres[]
    moods[]

  feedback
    canLike
    canDislike
    feedbackToken?
    expiresAt?

  metadata
    releaseDate?
    popularity?
    explicit?
    variantType?
```

Unknown values remain null/unknown rather than being guessed.

## 8. Identity normalization and deduplication

Recommendation origin and playback source are different concepts.

Candidate normalization should use the strongest available identity evidence in roughly this order, subject to safeguards:

1. direct known cross-provider mappings
2. ISRC
3. MusicBrainz recording identity
4. provider IDs mapped through Parachord/Achordion caches
5. normalized artist/title/album/duration matching
6. fuzzy fallback with confidence threshold

False-positive protection is essential for:

- live versions
- remasters
- covers
- acoustic versions
- radio edits
- explicit/clean variants
- sped-up/slowed versions

The existing Parachord resolver confidence model should be reused where appropriate instead of creating an unrelated second matching system.

## 9. Merge pipeline

Initial target pipeline:

```text
Recommendation Surfaces
          ↓
fetch independently / fail independently
          ↓
canonical candidate normalization
          ↓
identity matching + deduplication
          ↓
feature extraction
          ↓
filters
          ↓
source/surface score normalization
          ↓
ranking
          ↓
diversity constraints
          ↓
final unified result
```

One failing source must not block results from other sources.

## 10. Ranking principles

The engine must not assume that a track recommended by more services is automatically better.

Consensus is only one optional signal.

Possible ranking features:

- source weight
- surface weight
- normalized native source score
- source rank
- consensus count
- historical success of source/surface for this user
- track familiarity
- artist familiarity
- novelty
- popularity
- release age
- genre affinity
- mood affinity
- already played recently
- favorite status
- playlist membership
- user blocks
- diversity/repetition penalty
- recommendation reason/context

Example conceptual score:

```text
finalScore =
    sourceWeight
  + surfaceWeight
  + normalizedNativeScore
  + affinity
  + noveltyWeight
  + optionalConsensusBoost
  - repetitionPenalty
  - recentPlayPenalty
  - blocked/filtered candidates
```

The real implementation should avoid pretending this simple formula is universally correct. Ranking should be modular and testable.

## 11. User control

The user should be able to change what the recommender optimizes for.

### Simple presets

Examples:

- For You
- Discovery
- Safe/Familiar
- New Releases
- Track Radio
- Obscure

### Expert controls

Potential controls:

- enable/disable sources
- enable/disable individual surfaces
- source weights
- surface weights
- novelty preference
- familiarity preference
- popularity preference
- consensus weight
- already-heard penalty
- favorite inclusion/exclusion
- playlist-member inclusion/exclusion
- artist repetition limit
- genre inclusion/exclusion
- mood inclusion/exclusion
- release-age range
- live/remix/cover filtering
- explicit content filtering
- artist/track/label block lists

Custom profiles should eventually be saveable.

## 12. Feedback and adaptive ranking

Local behavior can become a ranking signal without requiring AI.

Potential feedback signals:

```text
immediate skip
short listen
completed listen
repeat play
favorite
unfavorite
dislike/block
add to playlist
remove from playlist
manual "more like this"
manual "less like this"
```

The engine may eventually learn source/surface effectiveness by context.

Example:

```text
Deezer Flow + electronic        strong
Deezer Flow + rock              neutral
ListenBrainz discovery + pop    weak
YT Music track radio            strong
```

Any adaptive ranking should be:

- inspectable
- resettable
- optional
- deterministic enough to test
- local-first unless the user explicitly enables external processing

## 13. Consensus

Consensus can be useful but must remain optional and bounded.

Example:

```text
Track A
  recommended by Deezer
  recommended by ListenBrainz
  recommended by Last.fm
```

This may increase confidence that the track is relevant, but it may also simply reflect popularity bias.

Therefore:

- consensus default weight should be modest
- user may reduce it to zero
- discovery presets may intentionally penalize over-common candidates
- obscure/novelty modes should not be dominated by consensus

## 14. Native provider surfaces vs recommendation-only plugins

Do not split every music service into a separate recommendation plugin unnecessarily.

Preferred model:

```text
DeezerProvider
  └─ RecommendationSource capability

YouTubeMusicProvider
  └─ RecommendationSource capability
```

Standalone recommendation systems can be plugins or native sources:

```text
ListenBrainz
Last.fm
future recommendation service
community algorithm
```

All implement the same logical source contract.

## 15. `.axe` recommendation capability

The current `.axe` system does not yet expose a generic recommendation-source contract. This is a planned extension.

Conceptual manifest addition:

```json
{
  "capabilities": {
    "recommendations": true
  }
}
```

Possible plugin methods:

```text
getRecommendationSurfaces()
getRecommendations(request)
submitRecommendationFeedback(event)
```

The exact API must be designed around:

- async runtime constraints
- cancellation
- timeouts
- authentication/secrets
- network permissions
- payload validation
- pagination
- caching
- source provenance
- versioned schema compatibility

Third-party plugins must never be trusted to return well-formed data.

## 16. Manual plugin installation

Separate but related goal:

- official marketplace remains available
- install `.axe` from local file
- install from URL
- optionally configure additional plugin repositories
- show source/trust provenance
- support hashes/signatures when practical
- reload plugins without requiring an app update

This enables third-party recommendation sources without expanding the core app for every experimental service.

## 17. Initial unified pools

Suggested first built-in pools:

### For You

Broad personalized recommendations. Can contain familiar and unfamiliar tracks.

### Discovery

Bias toward tracks/artists not already heavily represented in the user's history/library.

### Track Radio

Requires a seed track. Merge only related-track/radio surfaces.

### Artist Radio

Requires a seed artist. Merge only artist-related surfaces.

### New Releases

Merge personalized and editorial new-release surfaces.

### Replay

Known music the user may want to hear again. Kept separate from discovery.

Custom pools can be added later.

## 18. Initial provider/surface inventory

This is a planning list, not yet a statement that every endpoint is stable or legally usable. Each entry must be verified before implementation.

### ListenBrainz

- raw collaborative-filter recommendations
- Daily Jams
- Weekly Jams
- Weekly Exploration
- feedback

### Last.fm

- recommended station/feed
- similar tracks
- similar artists
- tags/taste signals

### Deezer

- Flow
- Track Mix
- Artist Mix
- personalized Home
- Made for Me / SmartTracklists where currently exposed
- personalized/editorial new releases

### YouTube Music

- Quick Picks
- Listen Again
- personalized Home shelves
- radio/watch queue
- related tracks
- moods/genres

### Apple Music

- personal recommendations
- recommended albums/playlists/stations
- Heavy Rotation
- Recently Played
- New Releases/editorial discovery

### Spotify

- user top tracks/artists
- recently played/library signals
- any recommendation functionality still permitted by current API access level

### SoundCloud

- related tracks
- related users/artists
- discovery surfaces

### TIDAL

- daily/personalized mixes where accessible
- discovery mix
- new-release mix
- editorial/home modules

### Qobuz

- similar artists
- album suggestions
- track recommendations
- discover/editorial modules

### Bandcamp

- fan/collection/discover signals where useful

### Local

- history
- favorites
- playlists
- skips/completions/replays
- local metadata similarity
- future optional acoustic analysis

## 19. Relationship to AI recommendations

AI is optional and should fit into the same high-level architecture rather than own the architecture.

```text
AI recommendation source
  → RecommendationCandidate[]
  → same filters/ranking/dedup pipeline
```

AI-specific context may include:

- listening history
- library
- top artists/tracks/albums
- explicit user prompt

But disabling every AI provider must leave the complete native recommendation system functional.

## 20. Upstream-friendly implementation sequence

The safest implementation sequence is:

1. capture existing behavior in tests
2. introduce models/contracts with no user-visible behavior change
3. migrate Last.fm and ListenBrainz behind the new source contract
4. represent their separate surfaces explicitly
5. preserve existing merged result using the generic merger
6. expose standalone surfaces
7. add generic merge pools
8. add filters/weights
9. add `.axe` recommendation support
10. add new providers such as Deezer/YT Music
11. add adaptive ranking only after the deterministic system is stable

This sequence makes the architectural work useful to upstream Parachord even if fork-specific providers are never accepted upstream.

## 21. Non-goals for the first implementation

Do not attempt all of these in the first PR/iteration:

- machine-learning recommender training
- opaque AI ranking
- every provider at once
- perfect cross-provider identity
- fully automatic adaptive weights
- replacing provider-native recommendation UIs
- forcing every surface into a merged pool

The first milestone is architectural: make recommendation sources and surfaces first-class, generic concepts while preserving today's behavior.
