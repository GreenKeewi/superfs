# superfs
building infinite storage for everyone.

**Superfs is a shared filesystem for people and AI agents.** The goal is to let them work with the same project files across devices and sandboxes, without keeping a complete copy everywhere.

## What it does

- Fetches files when they’re needed and caches frequently used files locally.
- Syncs changes so other devices and agents can see them.
- Keeps file history so a previous version can be restored.
- Detects overlapping edits and preserves conflicts for review.

Those are the product goals described in our earlier discussion; the site doesn’t publish enough technical detail to confirm how they’re implemented.

## How it could work

1. **A client on each machine** presents the project as a normal folder.
2. When a program reads a file, the client checks its local cache. If the file isn’t there, it fetches it from cloud storage.
3. When a file changes, the client uploads a new version and updates shared metadata.
4. Other clients learn that the file changed, then fetch the new version when needed.
5. If two clients edit from the same old version, Superfs keeps both changes. Safe edits, like changes to separate JSON keys, could be merged; overlapping edits should be flagged for review.

## Where Cloudflare R2 fits

R2 looks like a reasonable place to store the **file contents and version snapshots**. It supports the S3 API, so standard S3 tools and libraries can connect to it. [Cloudflare R2 S3 docs](https://developers.cloudflare.com/r2/get-started/s3/)

R2 should **not be the whole sync system**. It stores objects, but Superfs also needs a metadata and coordination layer to track each file’s current version, who changed it, and whether an upload was based on stale data.

A possible layout:

```text
Laptop / agent sandbox
  └─ Superfs client + local cache
       ├─ reads and writes files
       └─ syncs with Superfs API
            ├─ metadata database: paths, versions, permissions, conflicts
            └─ Cloudflare R2: file contents and snapshots
```

For uploads and downloads, the API could authorize a specific operation with a short-lived presigned URL, letting the client transfer the file directly to or from R2. [Cloudflare presigned URL docs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)

## Important concurrency detail

R2 is strongly consistent for object reads and writes, but if two clients write to the **same object key**, the last completed write wins. That alone won’t protect simultaneous edits. [Cloudflare R2 consistency docs](https://developers.cloudflare.com/r2/reference/consistency/)

So Superfs should give every file version an immutable key, such as `project/file.json/version-42`, and use its metadata service to decide which version is current. If a client tries to publish based on an outdated version, the service can preserve both versions and report a conflict instead of silently overwriting one.

## When to build what

- **First:** R2 storage, file versions, metadata records, and basic upload/download.
- **Next:** local cache and a client that exposes files as a folder.
- **Then:** change notifications, offline behavior, permissions, and conflict handling.
- **Later:** agent integrations, automatic safe merges, and richer history tools.

**Short version:** R2 can hold the bytes. Superfs still needs a client, cache, metadata service, and version-aware conflict logic to make it feel like one shared drive.
