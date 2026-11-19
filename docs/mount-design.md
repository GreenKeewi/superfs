# Superfs — Mount Design (stub)

Status: **stub**. This defines the contracts so modules can be built in parallel. Details are filled in during phase 2.
Related: [PRD](prd.md), [architecture](architecture.md) (§7 cache, §8 write path, §10 platform abstraction)

## 1. FUSE ↔ core interface contract

OS shims (FUSE, macFUSE, WinFsp) call **only** the core's `Vfs` trait. The core never calls OS APIs directly. Sketch (Rust, proposed):

```rust
pub trait Vfs: Send + Sync {
    // namespace
    fn lookup(&self, parent: Ino, name: &OsStr) -> Result<Attr>;
    fn getattr(&self, ino: Ino) -> Result<Attr>;
    fn setattr(&self, ino: Ino, changes: SetAttr) -> Result<Attr>;   // truncate, mtime, mode
    fn readdir(&self, dir: Ino, cursor: DirCursor) -> Result<DirPage>;
    fn create(&self, parent: Ino, name: &OsStr, mode: Mode) -> Result<(Attr, Fh)>;
    fn mkdir(&self, parent: Ino, name: &OsStr, mode: Mode) -> Result<Attr>;
    fn unlink(&self, parent: Ino, name: &OsStr) -> Result<()>;
    fn rmdir(&self, parent: Ino, name: &OsStr) -> Result<()>;
    fn rename(&self, from: (Ino, &OsStr), to: (Ino, &OsStr), flags: RenameFlags) -> Result<()>;

    // data
    fn open(&self, ino: Ino, flags: OpenFlags) -> Result<Fh>;
    fn read(&self, fh: Fh, offset: u64, len: u32) -> Result<Bytes>;
    fn write(&self, fh: Fh, offset: u64, data: &[u8]) -> Result<u32>;
    fn flush(&self, fh: Fh) -> Result<()>;      // per close(2) of a dup'd fd
    fn fsync(&self, fh: Fh, datasync: bool) -> Result<()>;
    fn release(&self, fh: Fh) -> Result<()>;    // last close

    fn statfs(&self) -> Result<StatFs>;
}
```

Contract rules:

- **Errors** are a core `Error` enum (`NotFound`, `Exists`, `NotEmpty`, `Io`, `NoSpace`, `PermissionDenied`, `Offline`, `Conflict`, …). Each shim maps them to errno or NTSTATUS. `Offline` maps to `EIO` on uncached reads.
- **Inodes** are stable for the lifetime of a mount: `(branch, path) → Ino`, with renames tracked. They are not persisted across remounts.
- **Threading:** every method may be called concurrently. The core does its own locking. Shims must not serialize calls.
- **Blocking:** methods may block on network I/O, bounded by timeouts. Shims run them on a worker pool, never on the kernel channel thread.
- **Case sensitivity:** the core exposes a `case_insensitive` capability. On macOS/Windows it rejects names that differ only in case (`Exists`).
- **TBD:** xattrs, symlinks (v1: store as small files with a flag), file locking passthrough, mmap coherence.

## 2. Write-path semantics

- `write()` goes to the **pending-write buffer** (a DIRTY generation, see architecture §8). It returns once the data is in the local buffer.
- **Upload on close / fsync.** The DIRTY generation is sealed on `release()` (last close) and on `fsync()`. `flush()` on non-final closes does not seal. Sealing starts upload → commit in the background.
- `close()` **does not wait** for the upload. It returns after the sealed snapshot is durable in the local journal.
- `fsync()` **waits for COMMITTED** (or CONFLICT) while online, bounded by a timeout (proposed 30 s). If offline or past the timeout, it returns after the journal is durable locally and logs a warning. *(Open question: return `EIO` instead, for strict callers?)*
- Atomic-save pattern (`write tmp` → `rename tmp target`): the core recognizes rename-over-existing and commits the content as a **new version of `target`**, with `base_version` = target's previous head, not as delete + create. This keeps history and attribution intact for editors.
- Truncate + rewrite within one open handle produces one version, not two.
- Conflicts (`409`) never surface as a write error. The user's data is preserved as a server-side version and as a local conflict copy (architecture §6).

## 3. Crash-consistency guarantees

| Guarantee | Scope |
|---|---|
| Data written before a successful `fsync()` or `close()` survives a client crash or power loss | Durable in the local journal and later committed |
| Server never sees a torn file | Versions are whole-file objects. A commit references only fully uploaded objects (sha256 verified). |
| Each committed version is exactly one sealed snapshot | No interleaving of two generations |
| Commits of one file are applied in generation order | Generation *k+1* is based on *k* |
| No silent loss on concurrent edit | CAS + conflict record |
| **Not guaranteed:** writes not yet followed by close/fsync | May be lost on crash, matching local-FS expectations |
| **Not guaranteed:** cross-file atomicity | Each file commits independently (no multi-file transactions in v1) |

Recovery on start: replay the journal → re-attach DIRTY buffers → resume UPLOADING (same `upload_id`) → retry UPLOADED commits → rebuild the in-memory metadata index from the metadata cache plus `files?since=<last cursor>`.

## 4. Metadata caching for OS traffic

Most filesystem calls are metadata calls from the OS and tools, not reads by the user. Handle them locally:

- **Attr / entry cache** with kernel TTLs (proposed 1 s attr, 1 s entry). Invalidation comes from the change feed, not from TTL expiry.
- **Negative lookup cache** for paths that do not exist (`.git/index.lock` probes, `node_modules` resolution, `__pycache__`). This matters a lot for `git status` and compilers.
- **Directory listings** are cached per `(branch, dir)` and refreshed from the feed. `readdir` never goes to the network for a dir seen within the session, unless the feed invalidated it.
- **OS junk stays local:** `.DS_Store`, `._*`, `.Spotlight-V100`, `.Trashes`, `Thumbs.db`, `desktop.ini`, `$RECYCLE.BIN`. These are stored only in a local overlay and never uploaded.
- **Indexers:** set `.metadata_never_index` (macOS) and the not-content-indexed attribute (Windows) at the mount root, so Spotlight/Windows Search don't pull the whole drive into the cache.
- **Thumbnail / preview reads** that only touch the first bytes are served block by block (1 MiB) and do not fetch whole files.
- All of this lives inside the ~50 MB metadata budget.

## 5. Module decomposition (parallel implementation)

Each module has one owner and a fixed interface. Modules talk only through the listed interfaces, so they can be built and tested independently with fakes.

| Module (crate) | Responsibility | Depends on (interfaces) | Testable with |
|---|---|---|---|
| `sfs-vfs` | Implements `Vfs`: inode table, handles, path resolution, overlay reads | `Cache`, `WritePath`, `MetaClient` | Fake cache/meta, in-proc tests |
| `sfs-cache` | RAM + disk block cache, LRU, checksums | `Budget`, `Transfer` | Temp dirs, fake transfer |
| `sfs-writepath` | DIRTY→…→COMMITTED state machine, generations, backpressure | `Journal`, `Transfer`, `MetaClient`, `Budget` | Fault-injecting fakes |
| `sfs-journal` | Append-only, fsync'd state log + replay | local FS | Crash/kill tests |
| `sfs-budget` | Single accountant for the hard 2 GB cap: reserve/release per pool | — | Property tests |
| `sfs-meta` | Metadata API client, metadata cache (SQLite), feed watcher | HTTP | Mock server |
| `sfs-transfer` | Presigned GET/PUT, multipart, retries, URL refresh | HTTP | Local S3 (MinIO) |
| `sfs-fuse` / `sfs-macfuse` / `sfs-winfsp` | Thin OS shims → `Vfs` | `Vfs` only | Fake `Vfs`, OS CI runners |
| `sfs-cli` | Agent CLI (no mount) on `sfs-meta` + `sfs-transfer` | `MetaClient`, `Transfer` | Integration tests vs staging |

Suggested order: `sfs-budget`, `sfs-journal`, `sfs-meta`, `sfs-transfer` (parallel) → `sfs-cache`, `sfs-writepath` (parallel) → `sfs-vfs` → shims (parallel per OS).

## 6. Open questions

- Strict vs best-effort `fsync()` when offline (§2).
- Rename of large directories: one metadata op server-side, or per-file versions?
- Per-mount branch switching (mount a fork at a second path vs switch in place).
- mmap write coherence across machines (likely: documented as unsupported).
