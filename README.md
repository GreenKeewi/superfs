# superfs
building infinite storage for everyone.

**Superfs is the filesystem humans and coding agents share.** People and AI agents work on the same project files across laptops, CI, and sandboxes, without keeping a full copy everywhere.

## Docs

- [Product requirements](docs/prd.md): vision, goals/non-goals, user stories, functional requirements, failure modes, metrics, build order, testing, risks
- [Architecture](docs/architecture.md): topology, R2 layout, metadata API, DB schema, forks, conflicts, cache, write path, auth, failure modes
- [Mount design (stub)](docs/mount-design.md): FUSE ↔ core contract, write-path semantics, crash consistency, module breakdown

## Three surfaces, one drive

| Surface | For | Phase |
|---|---|---|
| **Mount** (FUSE / macFUSE / WinFsp) | People and tools that expect a normal folder | 2 |
| **CLI / HTTP API** | Coding agents, scripts, CI | 1 |
| **S3-compatible endpoint** | Existing programs and S3 SDKs | 4 |

## Key decisions

- **Agents are first-class users.**
  - **Scoped, revocable tokens:** limited to path prefixes, permissions, a branch, and an expiry. Folder sharing is a scoped token.
  - **Per-version author attribution:** every version records which human or agent wrote it.
  - **Constant-time forks:** metadata-only, copy-on-write. Each agent task gets its own isolated drive.
  - **Change feed:** `files?since=T` lets agents react to edits instead of polling.
- **Nothing is silently overwritten.** Each version is an immutable R2 object (`path/v-N`). Commits are compare-and-swap. On a human-vs-agent conflict both versions are kept for review.
- **Disk cache is on by default.** Files are fetched lazily in 256 KB–1 MiB blocks and cached in RAM and on disk.
- **Fixed 2 GB maximum client footprint.** About 1.7 GB read cache, 250 MB pending writes, and 50 MB metadata, under a hard cap. Pending writes are never evicted. Writers get backpressure instead.
- **No user-facing tuning.** There are no cache size knobs or sync settings to configure.
- **R2 holds bytes. The metadata service holds truth.** Clients transfer data directly with short-lived presigned URLs.

## Build order

1. **CLI MVP:** metadata API, R2 versioned storage, sync engine, cache, CLI. Agents can use it immediately.
2. **Mount design doc:** internal interface and write-path semantics frozen before FUSE code.
3. **Mount:** per-OS bindings (FUSE / macFUSE / WinFsp) against the frozen interface.
4. **Forks + change feed**
5. **Tauri desktop app**
6. **Later:** S3-compatible endpoint, notifications, locks, safe merges, multi-user permissions
