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

**Confirmed, policy only (not a reader-visible format invariant)**: the Rust
reference implementation's writer-side minimum-input-size threshold below which
compression is skipped is `MIN_COMPRESSION_INPUT_BYTES = 256` bytes, defined once
and shared by both the SST block compression path and the WAL value compression
path (`wal.md` §4.3). This confirms it's a genuine writer heuristic as this
document already assumed — a reader must accept a compressed or uncompressed value
of any size regardless of this number, which exists purely to avoid paying
compression overhead on inputs too small to benefit. Recorded here for reference,
not as a format requirement; a conforming writer is free to use a different
threshold (or none) without breaking interop.
