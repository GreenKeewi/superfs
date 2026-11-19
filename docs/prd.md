# Superfs — Product Requirements

> **The filesystem humans and coding agents share.**

Status: draft · Owner: superfs core · Related: [architecture](architecture.md), [mount design](mount-design.md)

## 1. Vision

People and coding agents increasingly work on the same projects. They work from laptops, CI runners, cloud sandboxes, and ephemeral agent containers. Today each of those places keeps its own full copy of the project and syncs it by hand (git push/pull, rsync, zip uploads). Superfs replaces those copies with one shared, versioned drive:

- Every machine sees the same tree, fetched lazily and cached locally.
- Every change is a new immutable version, attributed to the human or agent who made it.
- Agents are first-class users. They get scoped, revocable tokens, their own forks to work in, and a change feed to react to.
- Conflicts between humans and agents are kept and surfaced, never silently overwritten.

## 2. Surfaces

Superfs has three ways in, all backed by the same metadata service and object store.

| Surface | Primary user | What it is | Phase |
|---|---|---|---|
| **Mount** | People (and tools that expect a real path) | A normal folder via FUSE (Linux), macFUSE (macOS), WinFsp (Windows). Lazy fetch, local cache, background upload. | 2 |
| **CLI / API** | Coding agents, scripts, CI | `superfs` CLI and HTTP API: read, write, history, restore, fork, watch. No kernel driver needed, so it works in locked-down sandboxes. | 1 |
| **S3-compatible endpoint** | Existing programs and SDKs | Any S3 client can read and write a drive (or a fork) by bucket/key. | 4 |

## 3. Users

- **Developer (human):** edits files in an IDE on one or more machines and expects them to look like local files.
- **Coding agent:** runs in a sandbox with limited permissions. It reads part of a project, makes changes, and hands results back for review.
- **Automation / program:** CI jobs, build systems, and data tools that talk S3 or HTTP.
- **Drive owner / admin:** controls who and what can access a drive, and can revoke access.

## 4. User stories

Priority: **P0** = required for first usable release; **P1** = required before general availability; **P2** = later.

### Agents as first-class users

| ID | Priority | Story | Acceptance criteria |
|---|---|---|---|
| A1 | **P0** | As a drive owner, I can mint a **scoped, revocable agent token** so an agent can access only what it needs. | `POST /tokens` returns a token with path-prefix scope(s), permissions (`read`, `write`, `fork`, `watch`), optional branch restriction, and expiry. Revocation takes effect on the next API call (≤ 5 s including cache TTL). Tokens are never able to mint broader tokens. |
| A2 | **P0** | As a reviewer, I can see **who authored every version** (which human or which agent token), so I can audit and attribute changes. | Every row in `versions` has a non-null `author` (user id or token id + display label). `history` and the change feed expose it. Author cannot be spoofed by the client. |
| A3 | **P1** | As an agent orchestrator, I can **fork a drive in constant time** for each agent task, so agents work in isolation and the result can be reviewed or discarded. | `POST /forks` returns in O(1) regardless of drive size (metadata only, no object copies). Reads on the fork fall through to the parent at the fork point. Writes stay on the fork. Forks can be listed, diffed against parent, merged, or deleted. |
| A4 | **P1** | As an agent or tool, I can subscribe to a **change feed** so I react to edits instead of polling the whole tree. | `GET /files?since=T` returns all changes after cursor `T` (path, version, author, branch, op). A streaming/long-poll variant exists. The cursor is monotonic and resumable. No changes are skipped across reconnects. |
| A5 | **P1** | As a developer, when an agent and I edit the same file, **conflicts are resolved explicitly** and neither side's work is lost. | Compare-and-swap commit detects stale `base_version`. Both versions are kept. A `conflicts` row records both sides and their authors. The mount shows a conflict copy. CLI/API expose `resolve` (pick ours/theirs/merged). Human edits are never auto-discarded in favor of agent edits. |
| A6 | **P1** | As a drive owner, I can **share a single folder** with a person or agent using token scopes, without exposing the rest of the drive. | A token scoped to `/docs/` cannot list, read, or infer the names of paths outside `/docs/`. Sharing a folder = minting a scoped token (read-only or read-write). |

### Core filesystem

| ID | Priority | Story | Acceptance criteria |
|---|---|---|---|
| F1 | **P0** | As a developer or agent, I can read and write files through the CLI/API. | upload-url → PUT to R2 → commit, and download-url → GET. Works with files up to 5 GB (multipart). |
| F2 | **P0** | As a user, I can view a file's **history** and **restore** any prior version. | `history` lists versions with author, size, timestamp, base version. `restore` creates a new version whose content equals the chosen one (history is append-only). |
| F3 | **P0** | As a user, my data is not silently lost or overwritten. | Immutable version objects. CAS commits. Crash-consistent client (see [mount design](mount-design.md)). |
| F4 | **P1** | As a developer, I can **mount** a drive as a normal folder on Linux, macOS, and Windows. | Standard editors, `git`, compilers, and `ripgrep` work on the mount. Repeated reads of cached files run at local-disk speed. |
| F5 | **P1** | As a developer, I can keep working **offline** with cached files and have my edits sync when I reconnect. | Pending writes persist on disk across restarts. Conflicts on reconnect follow A5. |
| F6 | **P2** | As a team, I can take **advisory locks** on files (e.g. binary assets). | `locks` table is reserved in phase 1 and its API ships in phase 4. |
| F7 | **P2** | As a program, I can use any **S3 SDK** against a drive or fork. | ListObjectsV2, GetObject, PutObject, multipart, HeadObject against `/{drive}[@branch]/key`. |
| F8 | **P2** | As an integrator, I can receive **notifications** (webhooks) on changes. | Built on the change feed. Filtered by path prefix and branch. |

## 5. Client footprint guarantee

Superfs must be safe to run on a laptop and inside small agent sandboxes.

- **Hard cap: the client never uses more than 2 GB of local storage + cache memory combined for superfs data.** The cap is fixed. There is **no user-facing tuning knob.**
- The disk cache is **on by default**.
- The internal budget split (see [architecture §7](architecture.md#7-client-cache)) is ~1.7 GB read cache, ~250 MB pending-write buffer, ~50 MB metadata cache.
- When the pending-write buffer is full, writers get backpressure: writes block until uploads drain. Unsynced data is never evicted.
- The CLI/API surface in sandboxes can run with a zero-byte cache (stream-only) but still never exceeds the cap.

## 6. Success metrics

| Metric | Target (GA) |
|---|---|
| Fork creation latency (p99), any drive size | < 100 ms |
| Cold open-to-first-byte for a 1 MB file (same region) | < 150 ms p50, < 500 ms p99 |
| Warm (cached) read throughput on mount | ≥ 80% of native local disk |
| Commit visible to other clients via change feed | < 2 s p95 |
| Token revocation effective | < 5 s p99 |
| Client local footprint | ≤ 2 GB, always (hard guarantee, monitored) |
| Silent data loss incidents | 0 |
| Versions with missing/incorrect author attribution | 0 |
| Agent adoption: share of commits authored by agent tokens | tracked, no target |
| Weekly active drives with ≥ 1 human and ≥ 1 agent author | primary growth metric |

## 7. Non-goals (for now)

- POSIX-perfect semantics (hard links, cross-file atomic rename across directories, mmap write-sharing between machines).
- Real-time collaborative editing (character-level CRDT). Superfs works at file-version granularity.
- Automatic semantic merges beyond simple, provably non-overlapping cases. These may come later.
- Being a git replacement. Superfs is the working tree layer. Git can live on top.

## 8. Roadmap

| Phase | Deliverable | Key stories |
|---|---|---|
| **1 — CLI** | Metadata API, R2 versioned storage, `superfs` CLI, scoped tokens, author attribution, history/restore | F1, F2, F3, A1, A2 |
| **2 — Mount** | Portable client core + FUSE/macFUSE/WinFsp shims, two-tier cache, write-path state machine, offline | F4, F5, A5, A6 |
| **3 — Forks** | Branches table, `POST /forks`, copy-on-write reads, diff/merge, change feed | A3, A4 |
| **3b — Tauri app** | Desktop app (Tauri) on top of the mount: sign-in, drive picker, conflict resolution UI, token management, sync status | A5, A6 (UI) |
| **4 — Platform** | S3-compatible endpoint, webhooks/notifications, advisory locks | F6, F7, F8 |

Order is strict: CLI → mount → forks → Tauri → S3 endpoint / notifications / locks. The Tauri app is a packaging and UX layer over the phase 2–3 client, so it is numbered 3b rather than a phase of its own.
