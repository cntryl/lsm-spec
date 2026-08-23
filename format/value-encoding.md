# Value Encoding (shared primitives)

Status: draft. This document holds encoding primitives that are logically
independent of any single on-disk structure but are referenced by more than one
format document (currently [`wal.md`](wal.md), and expected to be referenced by
`sst.md` once written). Keeping them here avoids two format docs defining the same
enum with a risk of drifting out of sync.

## 1. Compression algorithm IDs

A single byte identifying the compression algorithm applied to an opaque value
payload (a WAL record's value, an SST block, etc.). The byte's meaning is fixed
across every structure that embeds it:

| Value | Name | Notes |
|---|---|---|
| 0 | `None` | Payload is stored uncompressed. |
| 1 | `Lz4` | LZ4 (block format, not frame format). |
| 2 | `Zstd3` | Zstandard, compression level 3. |
| 3 | `Zstd9` | Zstandard, compression level 9. |

A decoder encountering an algorithm ID it does not implement must treat the value
as undecodable/corrupt rather than guessing — this byte is a hard compatibility
requirement, not a hint.

Whether a given payload *is* compressed at all, and by which algorithm, is recorded
by the containing structure (e.g. WAL TLV tag `COMPRESSION`, §4.2 of `wal.md`) —
this document defines only the shared meaning of the ID byte itself, not where it's
stored.

**TODO: verify** the exact writer-side size threshold below which compression is
skipped even when otherwise enabled. This is a writer heuristic (see `wal.md` §4.3),
not a reader-visible format invariant, and may belong in a future
`conformance/` note rather than here — recorded as open.
