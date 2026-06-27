# geeksFile

A distributed file system built from scratch — Reed-Solomon erasure coding,
AES-256-GCM encryption, intelligent placement, and Merkle verification —
with a live web dashboard and CLI.

🔗 **Live demo:** [Deployment on Render](https://geeksfile.onrender.com)

---

## What this is

geeksFile is a working distributed file system. Files uploaded through the web
UI or CLI are compressed, split into chunks, erasure-coded into encrypted
fragments, and placed across a cluster of storage nodes. A controller
orchestrates the pipeline and exposes a REST API; a Preact SPA visualises the
cluster as a live force-directed graph with per-stage pipeline telemetry.

The system is designed to survive node failure: any K fragments out of N can
reconstruct the original chunk, fragments are distributed across distinct
nodes by a CRUSH-style placement function, and end-to-end integrity is
verified via a Merkle tree on download.

This repository documents the architecture and design publicly while keeping
the implementation private.

---

## Architecture

\```
┌───────────────────────┐
│   Web UI (Preact SPA) │  ← Dashboard + Playground
│   :9000 (static)      │
└───────────┬───────────┘
            │
┌───────────▼───────────┐
│   Ground Controller   │  ← FastAPI (REST API + orchestration)
│   :9000               │
└───────────┬───────────┘
            │
  ┌─────────┼─────────┐
  │         │         │
┌─▼──┐   ┌─▼──┐   ┌──▼─┐   ┌────┐
│ N1 │   │ N2 │   │ N3 │   │ N4 │   ← Storage Nodes (FastAPI)
│8001│   │8002│   │8003│   │8004│      + dynamically added nodes
└────┘   └────┘   └────┘   └────┘
\```

Five independent layers — controller, storage nodes, placement, replication,
and UI — each replaceable without touching the others. Storage nodes can be
added or removed at runtime; the controller spawns and kills real processes.

---

## Upload Pipeline

\```
File bytes
  → LZ4 compress
  → Split into 256 KB chunks
  → SHA-256 CID per chunk
  → Reed-Solomon encode (K=2, N=adaptive)
  → AES-256-GCM encrypt each fragment (unique nonce per fragment)
  → CRUSH-Lite placement (N distinct nodes per group)
  → ThreadPoolExecutor parallel distribution
  → SHA-256 Merkle tree over chunk CIDs
  → SQLite manifest persistence
\```

## Download Pipeline

\```
File ID
  → Load manifest from SQLite
  → Check fragment LRU cache (512 items)
  → asyncio.gather() parallel fetch (K fragments per group)
  → AES-256-GCM decrypt
  → Reed-Solomon decode
  → Reassemble chunks
  → LZ4 decompress
  → Merkle tree verification
  → StreamingResponse (256 KB chunks)
\```

---

## What this demonstrates

- **Distributed storage primitives end-to-end.** Erasure coding, replication,
  fault-tolerant placement, cryptographic integrity — implemented, not
  imported as a black box.
- **A working control plane.** Dynamic membership (add/remove nodes at
  runtime via REST), health checks, per-node statistics, and event streaming.
- **Observability over invisibility.** The Playground view streams stage-by-stage
  pipeline telemetry — compression ratio, fragment count, placement choices,
  per-stage timing — making the internals of an upload visible in real time.
- **A real frontend, not a Swagger page.** Preact + signals + a force-directed
  D3 cluster graph that updates as the cluster changes.
- **A real CLI.** Six commands, Rich terminal output, useful for both
  demos and scripting.

---

## Technical Details

### Erasure Coding
- **K = 2** data shards, **N = adaptive** (4–6) total shards per chunk group
- Reed-Solomon over GF(2^8) via `zfec`
- Reconstructable from any K of N fragments per group
- Node-diverse placement: each fragment in a group lands on a distinct node

### Encryption
- AES-256-GCM authenticated encryption
- 256-bit random key per file (stored in manifest)
- 96-bit random nonce per fragment
- Format: `nonce(12 B) || ciphertext || tag(16 B)`

### CRUSH-Lite Placement
\```
score(fragment, node) = hash(fragment_key || node_id) × node_weight
node_weight       = available_mb / (1 + latency_factor)
\```
Group-level selection ensures all N fragments in a group are placed on N
distinct nodes, so fault tolerance matches RS redundancy.

### Merkle Verification
- Binary SHA-256 tree over ordered chunk CIDs
- Root hash stored in manifest, verified on every download
- Detects corruption, tampering, and incorrect chunk ordering

### Caching
- LRU fragment cache (512 items) on the controller for hot fragments
- Skips network round-trips on repeat reads

---

## REST API (selected)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/cluster` | GET | Cluster status (nodes, health) |
| `/upload` | POST | Upload file |
| `/upload-verbose` | POST | Upload with per-stage telemetry |
| `/download/{file_id}` | GET | Download file (streaming) |
| `/files` | GET | List all files |
| `/nodes` | POST | Add a node (spawns process) |
| `/nodes/{node_id}` | DELETE | Remove a node (kills process) |
| `/events` | GET | System event feed |

Seventeen endpoints total. Full surface available in the live demo.

---

## Stack

- **Backend:** Python 3, FastAPI, Uvicorn, SQLAlchemy, SQLite
- **Cryptography & coding:** PyCryptodome (AES-256-GCM), zfec (Reed-Solomon),
  LZ4 (compression), `hashlib` (SHA-256 + Merkle)
- **Frontend:** Preact, htm, signals, vanilla JS
- **CLI:** Typer, Rich
- **HTTP:** httpx (async client), multipart/form-data, streaming responses

---

## References

- DeCandia et al. (2007). *Dynamo: Amazon's highly available key-value store.*
  SOSP '07. — placement, replication, and quorum patterns.
- Plank, J. S. (1997). *A tutorial on Reed-Solomon coding for fault-tolerance
  in RAID-like systems.* — erasure coding foundations.
- Weil et al. (2006). *CRUSH: Controlled, scalable, decentralized placement
  of replicated data.* SC '06. — inspiration for the placement function.
- Merkle, R. C. (1988). *A digital signature based on a conventional
  encryption function.* CRYPTO '87. — Merkle tree verification.

---

## Status

Working prototype. Demonstrable end-to-end via the live deployment.
Implementation code is private; this repository captures the public-facing
architecture and design.

---

## License
Documentation, diagrams, and content in this repository are licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
The implementation code is not included in this repository.
