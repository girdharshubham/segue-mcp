# Segue — product overview

> Working document. We refine this together before writing code.
> `CLAUDE.md` holds the locked-in technical decisions; this `doc/` folder is where
> we think out loud about the product.

## One-liner

Segue builds harmonically-mixed, BPM-progressing DJ **set playlists** in the user's
Spotify account, so they appear in rekordbox and can be mixed on the FLX4.

## Why it exists (the wedge)

Spotify's `audio-features`/`audio-analysis` API is permanently deprecated (since
2024-11-27). **Spotify can no longer tell you a track's BPM, key, or energy.** That
data must come from outside (GetSongBPM). Segue owns the two things an LLM is bad at:

1. getting reliable BPM/key per track, and
2. the Camelot/BPM **math** of ordering a set so the joins are smooth.

The *curation* taste ("warm up, peak at 7, cool down") is what LLMs are good at.

## Phased build strategy

We deliberately build in two phases so the MCP surface stays the source of truth.

### Phase A — pure MCP server (no LLM inside Segue)  ← we are here

Segue is a bag of deterministic tools. The brain is the user's MCP client (Claude).
- ➕ Cheapest path; no model keys/cost/latency in Segue; matches the verified workflow.
- ➖ Quality depends on the client LLM; curatorial intent must pass through
  primitive tool params.

### Phase B — same service, plus an internal Akka agent

Once A is nailed down, we add an Akka `AutonomousAgent` *behind* a slimmer, higher-level
tool (e.g. `build_set(brief)`). Segue becomes **both**: still reachable as an MCP, but
now able to autonomously grind through enrichment holes / match retries without
round-tripping every decision back to the client.
- Buys: autonomous execution; works even with a "dumb" client.
- Costs: a model provider + API key inside Segue; resolves the "two brains" split
  (client = conversational intent; Segue agent = execution).

**Rule of thumb:** anything the Phase-B agent does must be expressible as Phase-A
tools first. A is the contract; B is an optional brain on top of it.

## Architecture (from CLAUDE.md)

```
com.segue.domain       → Track, CamelotKey, SetList; key→Camelot; compatibility cost;
                         BPM/harmonic sequencing engine. PURE, no Akka deps. The "brain".
com.segue.application  → TrackAnalysisEntity (KVE cache: bpm/key/camelot) + View;
                         SpotifyClient & GetSongBpmClient wrappers; set-building service.
com.segue.api          → MCP endpoint exposing the tools.
```

## Open product questions (to resolve here, in order of ripple)

1. **End-to-end flow.** What does one session look like, turn by turn? Which tools
   fire, in what order, and what does the user see at each step?
2. **Division of labor / the interface.** How does the client LLM express curatorial
   intent through primitive tool params? (target BPM range? curve enum? energy direction?)
3. **Enrichment holes.** What does Segue do with a track GetSongBPM can't analyze —
   drop, place-with-warning, ask, allow manual BPM/key entry?
4. **Inputs.** Where do candidate tracks come from — one playlist, liked songs, merged
   sources? Does the user pre-pick the pool, or does Segue also *select* from a larger pool?
5. **Output shape.** New playlist vs. reorder-in-place. Where do transition
   notes/warnings go — playlist description, chat, both?
6. **"BPM-progressing" semantics.** Monotonic ramp? Warm-up/peak/cool-down arc?
   Fully user-specified each time?

## Decisions log

_(empty — we fill this as we settle the questions above)_
