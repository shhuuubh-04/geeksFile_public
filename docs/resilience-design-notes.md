# Resilience Architecture
**Date:** Apr 5, 2026
**Status:** Design discussion only — no code changes implemented.

---

## Context

The current geeksFile implementation uses pure **RS(K=2, N=4)** with all
fragments stored on satellite storage nodes. The controller is stateless:
it holds the manifest database and an in-memory fragment cache, but no
authoritative fragment storage.

This note documents an alternative resilience model considered during
development.

---

## Alternative: Controller-as-Fragment-Holder

### Concept

Instead of distributing all N fragments across satellite nodes only, the
controller holds fragment `f0`, and each satellite node holds one of the
remaining fragments (`f1`, `f2`, ..., `fN`).

With RS(K=2), reconstruction requires any 2 of N+1 total fragments. Because
one of those always lives on the controller, the recovery guarantee becomes:

> **Controller + any 1 surviving node** = full file reconstruction.

### Benefits

- **Single-node recovery:** even if N-1 satellite nodes fail simultaneously,
  the controller plus one survivor can rebuild any file.
- **Narrative fit:** matches the satellite-style cluster framing — "ground
  control + any one surviving satellite can recover data."
- **Predictable overhead:** storage overhead ≈ `(1 + N) / K`, which is ~250%
  for N=4 (vs ~200% for the current pure-node design).

### Trade-offs

- **Statelessness lost.** The controller becomes critical infrastructure
  rather than a stateless coordinator. Its failure domain now includes
  authoritative data, not just routing and metadata.
- **Persistent fragment storage on controller.** Requires actual on-disk
  fragment store on the controller — not just the manifest DB and
  in-memory LRU cache.
- **Higher coupling.** Restarting or migrating the controller now involves
  fragment durability concerns, not just process restart.

---

## Current Implementation

The current design intentionally **keeps the controller stateless**:

- RS(K=2, N=4) with all fragments on satellite nodes
- Any 2 of 4 nodes can reconstruct
- Controller holds only the manifest DB and an LRU fragment cache
  (performance, not resilience)
- Controller failure does not reduce fault tolerance — only routing
  availability

---

## Path Forward

The controller-as-fragment-holder model is a viable **optional deployment
mode** behind a feature flag. The right decision depends on the use case:

| Scenario | Recommended Mode |
|----------|------------------|
| Production: cheap controller restart, scaling, statelessness | **Current (pure-node)** |
| Demo / hackathon: emphasise "maximum recovery with minimum survivors" | **Controller-fragment-holder** |
| Edge / disconnected operation with one strong central node | **Controller-fragment-holder** |

For now, the pure-node design ships. The fragment-holder mode is documented
here as a candidate extension, not a planned feature.
