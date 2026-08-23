# Write-Ahead Log (WAL) Format

Status: draft, extracted from two independent implementations (a Rust engine and a C#
engine) for cross-checking. Sections marked "TODO: verify" reflect points where the
two implementations agree today but the spec author could not find an explicit,
independent design doc confirming the choice was deliberate (as opposed to
implementation-shared incidental behavior).

## 1. Overview

The WAL is the durability mechanism for an LSM-tree storage engine. Every mutation
(and transaction-framing marker) that the engine intends to make durable before it is
reflected in the in-memory table is first appended to the WAL. On crash/restart, an
engine replays WAL contents to reconstruct in-memory state (memtables) up to the last
durable point, before serving reads or accepting new writes.

The WAL format described here defines two independent, layered concerns:

1. **Framing** — how variable-length byte payloads are packed into a WAL file/stream
   with integrity protection (§3).
2. **Payload encoding** — how a single logical WAL record (an operation on a key, or a
   transaction-control marker) is serialized to the bytes carried inside one frame
   (§4).

Everything else — when frames are flushed/fsynced, batching/group-commit policy,
whether writes are mirrored to cloud storage synchronously or asynchronously — is
**behavioral/policy**, not format, and is called out explicitly in §7 so implementers
don't mistake it for a wire-compatibility requirement.

### 1.1 File and segment layout

A WAL lives under a directory (`wal_dir`) that contains:

- **One active file**, always named `wal.log` for any directory written by a
  conforming current writer. This is the only mutable WAL file: frames are appended
  to it as writes occur. It is never uploaded/shipped to secondary storage as-is
  (only sealed segments are).
- **Zero or more sealed segment files**, one per rotation, named as a fixed-width
  decimal segment id with a `.wal` extension:

  ```
  {segment_id:020}.wal
  ```

  The width is 20 characters (zero-padded), matching the decimal digit count of
  `u64::MAX` (18446744073709551615), so that lexicographic (string) sort order of
  filenames equals numeric order of `segment_id`. This matters because object stores
  and many filesystem APIs only offer lexicographic listing.

  Example: segment id `42` → `00000000000000000042.wal`.

- When the active file is rotated (sealed), it is renamed/moved to the next segment
  id's filename and a new empty `wal.log` is started.

Segment ids are monotonically increasing per WAL instance (`writer_epoch`-scoped in
cloud-durability configurations, see §1.2).

A legacy/alternate naming pattern `wal_{segment:06}.log` is also recognized by
segment-id parsing (`wal_000123.log` → `123`). **This is not a second valid name for
the current active file** — the active file is identified strictly as `wal.log` per
the bullet above, and the recovery/replay order below (§1.1 "Recovery/replay order")
only ever looks for `wal.log` in that role — but a reader's segment-id-parsing
routine must still recognize this pattern when it appears, since it may be present in
directories written by older code. **Confirmed** against the Rust reference
implementation's current source: no code path anywhere ever *writes* a file matching
this pattern (there is no `format!("wal_...")` call in the codebase) — it exists
purely as a decode-side parsing accommodation. Whether it was ever actually produced
by a now-removed historical writer version (rather than being purely defensive
parsing for hypothetical/foreign input) isn't determinable from the current source
alone — **TODO: verify** against the C# reference implementation and/or git history.

#### Recovery/replay order

On startup, a reader/recovery process must process WAL files in this order:

1. Sealed segment files, in **ascending `segment_id` order**.
2. The active file (`wal.log`), if present, last.

Within a single file, records are read in the order the frames appear (append order).
Global logical ordering across the whole WAL is: sealed segments by ascending id, then
active file, each file read front-to-back.

### 1.2 Cloud-backed WAL (optional layer)

Deployments that mirror the WAL to an object store add a layer on top of the on-disk
format described here; it does not change the byte format of a frame or record, but it
does change what "durable" and "authoritative" mean. This layer is describable as:

- **Epoch-scoped immutable segment objects.** A sealed local segment file's bytes are
  uploaded verbatim to an object key that encodes both a writer fencing epoch and the
  segment id:

  ```
  wal/epochs/{writer_epoch:020}/{segment_id:020}.wal
  ```

  (the `wal/` prefix is a deployment-configurable root; `epochs/{epoch}/` and the
  zero-padded segment filename are the fixed parts of the layout).

  **This is a real constraint on what may be uploaded, not just a naming
  convention:** because the object key names exactly one `writer_epoch`, the byte
  range uploaded under it MUST NOT itself mix records from more than one
  `writer_epoch` — unlike a local on-disk file, which §6.4 point 3 explicitly
  permits to mix epochs. A writer preparing an upload must guarantee this, e.g. by
  validating the byte range before upload and refusing/splitting/deferring it if it
  isn't single-epoch (the exact mechanism is a writer implementation choice; only
  the invariant — one epoch per uploaded object — is normative here).

- **A lease-fenced publication catalog.** A single JSON object (**not the WAL frame
  format** — see §1.2.1) lists which uploaded segment objects are authoritative for
  recovery. An uploaded object existing in the object store is *not* sufficient for it
  to be replayed — only entries present in the catalog are recovered. This lets an
  uploader whose lease was already revoked finish uploading without corrupting
  recovery: the orphaned object is simply never cataloged.
- **Fencing.** The catalog carries a monotonically increasing `fencing_epoch`. Catalog
  mutations (publish/retire a segment) are rejected unless the caller's writer epoch
  equals the catalog's current fencing epoch. Advancing the fencing epoch to a newly
  acquired lease is a precondition for recovery to run, and is separate from the
  runtime being allowed to advance its durability frontier from a cloud
  acknowledgment.

This two-tier design (immutable content-addressed-by-id segment objects + a small
authoritative index) is the general contract; the exact transport/API used to store
the catalog object (e.g., conditional PUT semantics) is object-store-specific and out
of scope for this document.

#### 1.2.1 Publication catalog encoding (JSON, not binary framing)

The catalog is encoded as JSON (pretty-printed) with this logical shape:

```jsonc
{
  "format_version": 1,
  "fencing_epoch": <u64, non-zero>,
  "segments": {
    "<segment_id as string>": {
      "segment_id": <u64>,
      "writer_epoch": <u64>,
      "max_sequence": <u64>,
      "size_bytes": <u64>,
      "content_crc32c": <u32>,
      "object_key": "<string, must equal the canonical epoch-scoped key for (segment_id, writer_epoch)>"
    },
    ...
  }
}
```

`<segment_id as string>` is the **canonical, unpadded decimal** representation of the
u64 `segment_id` (e.g. `"42"`, not `"00000000000000000042"`) — this is a different
string form than the zero-padded 20-digit segment id used in local segment filenames
and inside `object_key` above; do not confuse the two.

Validity constraints on decode:
- `format_version` must equal the currently understood version (`1`).
- `fencing_epoch` must be non-zero.
- Every entry's `segment.writer_epoch` must be `<= fencing_epoch`.
- Every entry's map key must exactly equal the canonical unpadded-decimal string form
  of that same entry's own `segment_id` field — a mismatch (e.g. key `"42"` on an
  entry whose `segment_id` is `99`) is corruption, not a warning.
- Every entry's `object_key` must exactly equal the canonical
  `wal/epochs/{writer_epoch:020}/{segment_id:020}.wal` key derived from its own
  `segment_id`/`writer_epoch` — a mismatch is corruption, not a warning.
- A segment's declared `size_bytes` and `content_crc32c` are a whole-object checksum
  (CRC32C over the entire uploaded segment byte stream) used to validate a downloaded
  object before trusting it for recovery, on top of the per-frame CRC32C described in
  §3.

This JSON schema is a control-plane detail specific to cloud-mirrored deployments.
Engines that do not support cloud WAL durability need not implement it, but if they
do, this is the compatibility surface.

## 2. Terminology

| Term | Meaning |
|---|---|
| Frame | The physical unit written to a WAL file: a fixed-size header + a variable-length payload, with an integrity checksum. |
| Payload | The bytes carried by one frame. Frame concerns (length, checksum) are orthogonal to what the payload bytes mean. |
| Record | A decoded logical WAL entry: one operation (or transaction marker) with its associated metadata. |
| Sequence number (`seq`) | A monotonically increasing u64 assigned by the writer to order records for a given engine instance/epoch. |
| Writer epoch | A monotonically increasing u64 fencing token stamped by the current writer. Used to detect and discard stale writes left behind by a superseded writer (e.g. after leader failover). |
| Segment | One rotated (sealed), immutable WAL file. |

## 3. Frame format (physical framing layer)

Every record is written to the WAL file as a length-prefixed, checksummed frame:

```
+------------------+------------------+----------------------------+
| payload_len (u32)| crc32c (u32)     | payload (payload_len bytes)|
| offset 0..4       | offset 4..8      | offset 8..(8+payload_len)  |
+------------------+------------------+----------------------------+
```

| Field | Size | Byte order | Description |
|---|---|---|---|
| `payload_len` | 4 bytes | little-endian | Length in bytes of the payload that follows. |
| `crc32c` | 4 bytes | little-endian | CRC32C (Castagnoli polynomial) checksum of the payload bytes only (header itself is not covered). |
| `payload` | `payload_len` bytes | — | Opaque to the framing layer; see §4 for its structure. |

Constants:

- Frame header length: **8 bytes**, always (`4 + 4`).
- Maximum payload length: **64 MiB (`64 * 1024 * 1024` bytes)**. A frame header
  declaring a larger length must be treated as corruption, not merely rejected as
  "too big to allocate" — this cap is a format invariant enforced on both the encode
  and decode paths.
- Checksum algorithm: **CRC32C** (i.e., CRC-32 with the Castagnoli polynomial,
  the same variant used by iSCSI/ext4/etc.) — not the classic CRC-32 (zlib/gzip)
  polynomial. This choice matters for interop: a decoder using the wrong polynomial
  will silently fail all checksum verification.
- The checksum protects only the payload; the header (length + checksum fields
  themselves) is not separately checksummed. A corrupted `payload_len` that still
  happens to point at a byte range whose CRC matches by chance is a residual risk
  common to length-prefixed logs; both known implementations accept this trade-off
  (no header CRC / no magic number in the frame header — see §3.1).

### 3.1 No magic number or version at the frame level

The 8-byte frame header carries no magic bytes and no version tag of its own —
versioning lives one layer up, in the payload encoding (§4). A frame is *structurally*
valid whenever `payload_len <= 64 MiB` and the trailing bytes checksum correctly; the
frame layer alone cannot distinguish "valid frame of an unknown future format" from
"corrupt frame," which is why the payload-level magic/version (§4.1) is the primary
forward-compatibility mechanism, and why recovery logic that needs to search for a
resynchronization point (§5.3) scans for the payload magic, not a frame magic.

**TODO: verify** whether a future format revision should add a frame-level magic/
version to allow multiplexing incompatible payload encodings within one physical
frame stream, or whether the payload-level magic is considered sufficient
permanently. Both current implementations use the no-frame-magic design; this is
recorded as intentional-by-consensus rather than as an oversight, but it has not been
tested against a real forward-incompatible payload version bump.

### 3.2 Zero-length payloads

`payload_len = 0` is representable by the frame format (an 8-byte frame with an empty
payload, whose CRC32C is the CRC32C of zero bytes). Whether any current payload
encoding ever legitimately produces a zero-length payload is a payload-layer question,
not a framing one — see §4.

## 4. Payload encoding (record layer)

### 4.1 Single-record payload: magic + version + TLV

A frame's payload (for anything other than an atomic transaction batch — see §4.4 for
that variant nested inside this same envelope) is encoded as:

```
+--------+---------+----------------------------------+
| magic  | version | TLV-encoded fields (repeated)     |
| 2 bytes| 1 byte  |                                    |
+--------+---------+----------------------------------+
```

| Field | Size | Value |
|---|---|---|
| `magic` | 2 bytes | ASCII `"MW"` (`0x4D 0x57`) |
| `version` | 1 byte | `1` (current) |

Total fixed prefix: **3 bytes**.

Following the 3-byte prefix, the remainder of the payload is a sequence of
**TLV (tag-length-value)** fields:

```
+-------+---------------+------------------+
| tag   | len (u32, LE) | value (len bytes)|
| 1 byte| 4 bytes       |                  |
+-------+---------------+------------------+
```

TLV header is 5 bytes (`1 + 4`), followed by `len` bytes of value.

**Decode rules:**
- Fields may appear in any order (though a canonical writer emits them in the order
  listed in §4.2).
- **Unknown tags are skipped** — this is the forward-compatibility mechanism: an older
  reader encountering a payload written by a newer writer that adds a new optional TLV
  tag can still decode all tags it understands and ignore the rest, provided the
  `magic`/`version` prefix is otherwise identical.
- **Duplicate tags are accepted; the last occurrence wins.** (This is a defined
  behavior in at least one implementation's doc comment; **TODO: verify** it is relied
  upon anywhere, versus simply being a permissive default that no writer exercises.)
- A decoder must reject the whole payload as corrupt if:
  - it is shorter than the 3-byte prefix,
  - `magic` doesn't match `"MW"`,
  - `version` doesn't match a value the decoder supports (only `1` is currently
    defined — there is no defined negotiation/migration behavior for a future
    version bump beyond "reject"),
  - a TLV header is truncated (fewer than 5 bytes remain when one is expected),
  - a TLV's declared `len` exceeds the remaining payload bytes,
  - any of the fields marked **required** below is absent after the full scan.

### 4.2 TLV tag registry

| Tag (decimal) | Name | Required? | Value type / size | Semantics |
|---|---|---|---|---|
| 1 | `OP` | yes | 1 byte | Operation kind, see §5.1. Must decode to a known `WalOpKind`; unknown byte value is corruption. |
| 2 | `CF_ID` | yes | 4 bytes, u32 LE | Column-family / keyspace identifier the operation applies to. |
| 3 | `SEQ` | yes | 8 bytes, u64 LE | Sequence number assigned to this record. |
| 4 | `KEY` | yes | variable | Key bytes (or range-start for a range-delete). May be zero-length. |
| 5 | `VALUE` | no | variable | Value bytes for a value-write operation. Presence (even of a zero-length value) is semantically distinct from absence — see §5.1. When compressed (see tag 9), holds the compressed bytes. |
| 6 | `EXPIRATION` | no | 8 bytes, u64 LE | Optional TTL/expiration timestamp, Unix epoch milliseconds. |
| 7 | `RANGE_END` | no | variable | Exclusive end key for a range-delete operation. |
| 8 | `TXN_ID` | no | 8 bytes, u64 LE | Transaction identifier, present when the record is part of a transaction. |
| 9 | `COMPRESSION` | no | 1 byte | Compression algorithm applied to `VALUE`, if any. Algorithm ID meanings are defined once, shared with other formats (e.g. SST blocks), in [`value-encoding.md`](value-encoding.md). |
| 10 | `WRITER_EPOCH` | yes | 8 bytes, u64 LE | Fencing epoch of the writer that produced this record. Required on every record (not just transactional ones). |

All multi-byte integers in TLV values are **little-endian**.

Both implementations examined require `OP`, `CF_ID`, `SEQ`, `KEY`, and
`WRITER_EPOCH` unconditionally; everything else is optional and its presence/absence
carries meaning per §5.

### 4.3 Encoding notes (non-normative but relevant to compatibility)

- A writer that has a value but wants to compress it may replace the raw value bytes
  with a compressed form under tag 5 and record the algorithm under tag 9. A reader
  that doesn't understand a given compression byte cannot safely decode the value —
  this makes the compression scheme part of the effective compatibility surface even
  though it's logically a value-codec concern layered on top of the WAL TLV format.
  The threshold for "worth compressing" is a minimum input size — confirmed as
  `MIN_COMPRESSION_INPUT_BYTES = 256` bytes in the Rust reference implementation,
  shared with the SST compression path (see
  [`value-encoding.md`](value-encoding.md#1-compression-algorithm-ids)); this
  threshold is a writer-side heuristic, not a format requirement — a reader must
  accept both compressed and uncompressed values regardless of size.
- Presence of `VALUE` as an empty (zero-length) TLV is a legitimate, distinct wire
  state from `VALUE` being entirely absent (used to represent "Put with an empty
  string value" vs. "Delete/no value").

## 5. Record types and semantics

### 5.1 Operation kinds (`OP` tag values)

| Value | Name | Class ("role") | Semantics |
|---|---|---|---|
| 0 | `Put` | value write | Upsert `KEY` → `VALUE` at `SEQ`. `VALUE` required. |
| 1 | `Insert` | value write | Same wire/replay semantics as `Put` at the WAL layer; the distinction (e.g. insert-only conflict semantics) is an engine-level API concern, not a WAL-format concern — the WAL treats `Put` and `Insert` identically during replay. |
| 2 | `Delete` | point delete | Tombstone `KEY` at `SEQ`. `VALUE` absent. |
| 3 | `DeleteRange` | range delete | Tombstone the half-open range `[KEY, RANGE_END)` at `SEQ`. `RANGE_END` required. |
| 4 | `TxnBegin` | transaction-begin marker | Marks the start of a (legacy-style, non-atomic-frame) transaction identified by `TXN_ID`. Carries no direct mutation. |
| 5 | `TxnCommit` | transaction-commit marker | Marks the transaction identified by `TXN_ID` as committed; on replay this is the trigger to apply all operations buffered since the matching `TxnBegin`. Carries no direct mutation of its own. |
| 6 | `TxnBatch` | atomic transaction batch | A single physical frame whose `VALUE` holds an entire transaction's operations, nested-encoded (§5.4). Carries no direct mutation itself — replay decodes and applies the nested batch. |

Every operation kind (including markers) carries `SEQ`, `CF_ID` (markers conventionally
use a zero/default CF), and `WRITER_EPOCH` per the required-field rule in §4.2.

`is_transaction_marker` = `{TxnBegin, TxnCommit, TxnBatch}`. These three carry no
direct memtable mutation on their own record; their effect is defined entirely by
§5.3–§5.4.

### 5.2 Sequence numbers and ordering

- `SEQ` is a per-engine-instance u64 that must be monotonically non-decreasing across
  the logical WAL order defined in §1.1 (sealed segments ascending, then active file,
  each front-to-back). Recovery uses the maximum observed `SEQ` to restore the
  engine's sequence counter.
- Sequence numbers are not required to be contiguous at the frame level in general,
  **except** within a transaction batch, where they must be exactly contiguous (see
  §5.4).

### 5.3 Transaction semantics

Two encodings for grouping multiple operations into one all-or-nothing unit exist in
the wire format, and a recovering reader must support both:

**(a) Legacy split-marker style** (`TxnBegin` … mutation records … `TxnCommit`):
- A `TxnBegin` record with a given `(writer_epoch, txn_id)` opens a transaction.
- Ordinary mutation records (`Put`/`Insert`/`Delete`/`DeleteRange`) that carry a
  matching `TXN_ID` are considered part of that open transaction and **must be
  buffered, not applied, until the matching `TxnCommit` is observed**.
- On `TxnCommit`, all buffered operations for that `(writer_epoch, txn_id)` are
  applied, in the order they were originally appended.
- If a `TxnCommit` for a `txn_id` is never observed (WAL ends, or the engine crashed
  before committing), the buffered operations for that transaction are **discarded** —
  never applied. This is what gives the transaction its atomicity/all-or-nothing
  property under torn-tail recovery.
- A duplicate `TxnBegin` for an already-open `(writer_epoch, txn_id)` is corruption.

**(b) Atomic single-frame batch style** (`TxnBatch`):
- The entire transaction is encoded once, as the `VALUE` of a single `TxnBatch`
  record, using the nested format in §5.4.
- Because it is one physical frame, it is atomic with respect to torn-tail truncation
  by construction: either the whole frame's CRC verifies (in which case the whole
  transaction is valid and is applied entirely) or it doesn't (in which case, per §6,
  it is either the very last frame in the active file — treated as an incomplete tail
  and dropped — or it is mid-stream corruption, a hard error).
- This is the preferred/current encoding for ordinary-sized transactions; legacy
  split-marker transactions remain supported for reading older data, **and are also
  still an actively-emitted current-writer path, not merely legacy-read
  compatibility**: the Rust reference implementation falls back to split-marker
  framing (`TxnBegin` / individual ops / `TxnCommit`) for a transaction too large to
  buffer as a single atomic `TxnBatch` frame (a "spill" path). A from-scratch writer
  that only ever emits `TxnBatch`, and treats split-marker purely as a read-path
  concern, will diverge from the reference implementation's behavior on oversized
  transactions.

Nested transaction markers (a batch containing another `TxnBegin`/`TxnCommit`/
`TxnBatch`) are invalid and must be rejected.

### 5.4 Nested transaction batch payload (inside a `TxnBatch` record's `VALUE`)

The `VALUE` field of a `TxnBatch` record (tag 5, itself a TLV value inside the outer
§4.1 envelope) holds its own self-describing structure:

```
+--------+---------+---------+-----------+-----------+----------+------------------+
| magic  | version | txn_id  | begin_seq | commit_seq| op_count | records...       |
| 2 bytes| 1 byte  | 8 bytes | 8 bytes   | 8 bytes   | 4 bytes  |                  |
+--------+---------+---------+-----------+-----------+----------+------------------+
```

| Field | Size | Byte order | Notes |
|---|---|---|---|
| `magic` | 2 bytes | — | ASCII `"TB"` (`0x54 0x42`) |
| `version` | 1 byte | — | `1` (current) |
| `txn_id` | 8 bytes | LE u64 | Must equal the outer record's `TXN_ID`. |
| `begin_seq` | 8 bytes | LE u64 | Sequence number of the (virtual) transaction start. |
| `commit_seq` | 8 bytes | LE u64 | Must equal the outer record's `SEQ`, and must be `> begin_seq`. |
| `op_count` | 4 bytes | LE u32 | Number of nested operation records that follow. Must equal `commit_seq - begin_seq - 1` (i.e., each nested op consumes exactly one sequence number, and `commit_seq` itself occupies the final one). Must be non-zero — an empty transaction batch is invalid. |

Followed by exactly `op_count` nested operation records, each encoded as (not TLV —
this is a dedicated fixed+length-prefixed layout, distinct from §4.1's TLV scheme):

```
+-----+--------+---------+-----------+---------------------------+-----------------------------+---------------------------------+
| op  | cf_id  | seq     | exp_flag  | [exp: u64]  key (len+bytes)| value: flag + [len+bytes]   | range_end: flag + [len+bytes]   |
|1byte|4 bytes |8 bytes  | 1 byte    |                            |                              |                                  |
+-----+--------+---------+-----------+---------------------------+-----------------------------+---------------------------------+
```

Field-by-field:
- `op` (1 byte): must be a mutation kind (`Put`/`Insert`/`Delete`/`DeleteRange`); a
  nested transaction marker here is invalid.
- `cf_id` (4 bytes, LE u32).
- `seq` (8 bytes, LE u64): must equal `begin_seq + 1 + index` where `index` is this
  record's zero-based position in the batch — i.e., **sequence numbers inside a batch
  must be exactly contiguous**, no gaps, no reordering.
- expiration: 1-byte flag (`0`/`1`) followed by an 8-byte LE u64 only if the flag is 1.
- `key`: 4-byte LE u32 length prefix followed by that many raw bytes (always present,
  even if zero-length).
- `value`: 1-byte flag followed by, if `1`, a 4-byte LE u32 length + bytes.
- `range_end`: 1-byte flag followed by, if `1`, a 4-byte LE u32 length + bytes.

Minimum length per nested record: 1(op) + 4(cf_id) + 8(seq) + 1(exp flag) +
4(key length prefix) + 1(value flag) + 1(range_end flag) = **20 bytes**, before
accounting for actual key/value/range-end content. The `value` and `range_end`
presence flags are each a mandatory 1 byte regardless of whether their optional
length+bytes follow (see the field table above) — both must be included when
computing this minimum. A decoder must use this minimum to bound `op_count` against
remaining bytes *before* attempting a per-record allocation, to avoid a small buffer
causing an attempted huge allocation from an attacker-/corruption-controlled
`op_count`. (**Confirmed**: the Rust reference implementation's own minimum-record-
length constant is computed as this same 7-term sum and equals 20, used for the
identical anti-huge-allocation guard.)

Trailing bytes after the last nested record (i.e. input not fully consumed) is
corruption.

Cross-checks a decoder must perform against the **outer** record (in addition to the
inner-payload self-consistency above):
- outer `TXN_ID` == inner `txn_id`
- outer `SEQ` == inner `commit_seq`

## 6. Recovery semantics: valid vs. torn/corrupt tail

Recovery/replay walks frames sequentially from byte 0 of a file (see §1.1 for
cross-file order) and must classify any failure to parse the *next* frame into one of
two buckets, because they are handled completely differently:

### 6.1 "Incomplete tail" (tolerable, only at the very end of the *active* file)

Conditions that indicate a writer was interrupted mid-append (crash, power loss)
partway through writing the *last* frame, rather than mid-stream corruption:

- Fewer than 8 bytes remain for a frame header at the current position.
- The frame header bytes read as all-zero **and** every remaining byte in the file
  from that position onward is also zero (a "zero-filled tail" — consistent with a
  pre-allocated/zero-initialized file region that a partial write never reached).
- The frame header's declared `payload_len` would require more bytes than remain in
  the file (a torn payload) — **provided** that scanning the remaining bytes for
  another independently-verifiable frame (payload magic matches, and its CRC
  verifies, and it decodes) does **not** find one. (See §6.3 — this guards against
  silently accepting a *corrupted length field* that happens to swallow real,
  verifiable data after it.)

**Handling:** When one of these conditions is hit while replaying the file identified
as the *final active* file in the replay order (§1.1) — i.e. `wal.log`, not a sealed
segment — the reader stops replay at the last verified frame boundary and treats
everything from there to EOF as if it were never written. This is not reported as an
error to the caller (though implementations may log it): a truncated tail on the
active file is an expected consequence of a crash between "frame partially written"
and "next fsync," and is exactly what WAL replay exists to tolerate.

If the same condition occurs in a **sealed segment file** rather than the final active
file, it is **not** tolerated — sealed segments are supposed to be immutable and
complete once rotated, so an incomplete tail there indicates real corruption (or a
segment file mistakenly still open for writes) and must be surfaced as an error
(subject to §6.2's salvage policy, if enabled).

### 6.2 True corruption (never silently tolerated)

Anything that isn't classifiable as an "incomplete tail" per §6.1 — a CRC mismatch, an
invalid payload magic/version, a malformed TLV, an operation code out of range, a
`payload_len` that overflows an offset computation, mismatched transaction batch
metadata, etc. — is corruption. Two supported reader policies for what happens next:

- **Strict**: propagate the error; recovery fails outright. This is the default/safe
  policy.
- **Salvage-valid-prefix**: stop replay at the point corruption was detected, keep
  everything successfully verified before it, and continue operating as if the log
  ended there (recording that corruption was observed, for observability/alerting).
  This is an explicit opt-in policy for tolerating partial data loss in exchange for
  availability; it must not be the default, and a caller choosing it is accepting a
  potential silent gap in durability from the corruption point onward.

Both policies must, before discarding anything, distinguish "corrupt length field that
happens to hide real recoverable data right after it" (§6.3) from "corrupt length
field with nothing salvageable after it" — the former is escalated to a hard error
even under Strict-adjacent handling of the final-active-file tail case, because
silently dropping *verifiable* trailing data is worse than failing loudly.

### 6.3 Verified-suffix detection (guards against a corrupted length field)

Because §6.1's "incomplete tail" tolerance exists specifically to accept *legitimate*
partial writes, a reader must guard against a corrupted `payload_len` field being
misclassified as a benign incomplete tail when in fact real, later, independently
verifiable data exists past where the (corrupt) declared length would place the next
frame boundary. The check: scan the undecodable remainder of the buffer for any byte
offset at which the payload-level magic+version prefix (`"MW"` + version byte, §4.1)
appears, such that a frame header immediately preceding it decodes with a matching
`payload_len`/CRC and the payload itself parses successfully. If such a frame is
found, the original truncation is **not** benign — it must be reported as a hard
error ("frame length overruns EOF and hides a verified later frame") rather than
silently truncating the log before recoverable data.

This check is O(n) over the ambiguous tail region only (bounded by the 64 MiB max
frame size, §3), not the whole file.

### 6.4 Writer-epoch fencing during replay

Because a single WAL directory can (in failover scenarios) contain records from more
than one writer generation:

1. Recovery performs an initial pass to discover, per file in replay order, the
   lowest sequence number and lowest replay-ordinal position at which each observed
   `writer_epoch` first appears. `writer_epoch == 0` is treated as unfenced/not
   participating in this mechanism. **Confirmed** against the Rust reference
   implementation: `writer_epoch == 0` is deliberately special-cased as an
   epoch-fencing-disabled sentinel in exactly the two places that matter (recording
   a frontier, and testing staleness) — not an incidental unset-field default.
2. A record from writer epoch *E* is considered **stale** (skipped, not applied,
   still counted separately in recovery stats) if there exists some other epoch
   *E' > E* whose first-seen sequence number is `<=` this record's sequence number,
   or whose first-seen replay-ordinal is earlier than this record's ordinal. In
   effect: once a higher epoch's writes are known to have started, any lower-epoch
   record whose effect could conflict with or be superseded by them is dropped. The
   "ordinal" is a single counter spanning the whole logical replay stream (all
   sealed segments in ascending order, then the active file), never reset between
   files — confirmed against the reference implementation.
3. **Epoch-mixing within a single physical local WAL file (active or sealed) is
   normal, not corruption, and must not be rejected.** This format's active file is
   reopened in append mode across a writer failover, not rotated at the epoch
   boundary — there is no mechanism in this document (see §1.1) that seals `wal.log`
   at the moment a new writer acquires a higher epoch. It is therefore the expected
   result of an ordinary crash-then-failover cycle for one `wal.log` — and, since
   rotation moves the file's bytes verbatim into a sealed segment, potentially a
   sealed segment too — to contain records spanning more than one `writer_epoch`,
   distinguished only by the per-record `writer_epoch` field, never by file
   boundaries. Point 2's staleness rule is the *sole* mechanism this document
   defines for resolving such a file during local recovery. A conforming reader
   MUST NOT implement a separate "reject on epoch-mixing" check for local recovery:
   doing so would fail on data this format's own append-on-reopen recovery model
   produces routinely, rejecting perfectly ordinary post-failover WAL state as
   corrupt. (This is narrower than, and does not contradict, the cloud-upload
   requirement in §1.2 that an *uploaded* object's byte range be single-epoch —
   that's a constraint on what a writer chooses to upload, not on what a local file
   may contain.)

   *(Spec correction: an earlier draft of this document stated the opposite here —
   that a file mixing more than one epoch is corruption. That was wrong, not merely
   unconfirmed: it directly contradicts the append-on-reopen recovery model this
   same document describes elsewhere, and enforcing it as written would break
   ordinary failover recovery. Confirmed against the Rust reference implementation:
   its primary local-recovery path correctly tolerates epoch-mixing via staleness
   alone and is proven correct by a passing test that interleaves two epochs in one
   file and replays successfully — that behavior is what this document now
   specifies. The reference implementation does separately enforce single-epoch
   validation in two narrow helpers reachable only from the cloud-upload/download
   path, which is the real, narrower requirement now captured in §1.2. The C#
   engine independently corroborates the same design — it also reopens its active
   WAL file in place across a failover rather than rotating it, with no
   epoch-mixing rejection anywhere in its local recovery path.)*
4. `max_epoch_seen` (the highest writer epoch observed anywhere during recovery) is
   surfaced to the caller so the engine can resume issuing new writes starting at an
   epoch higher than anything seen.

This fencing behavior is squarely a **format/replay-contract** matter (it changes
which records a compliant reader must apply), not merely internal engine policy — an
engine that ignores it would apply stale writes after a failover and violate the
durability/consistency contract implied by `WRITER_EPOCH` being a required field.

## 7. Format vs. policy — what's in scope here

The following are explicitly **behavioral/policy**, not part of the on-disk format
contract, and are not required for wire compatibility between two engines that both
correctly implement §3–§6:

- **Durability policy** (when to fsync: per-write "Strict", time/size-batched
  "Batched", "CloudMirrored", "CloudAsync", or fsync-free "BestEffort"). None of these
  change a single byte on disk — they only change *when* bytes already described by
  this spec are flushed/synced, and what a caller waits for before being told a write
  succeeded. A conforming reader must be able to replay a WAL regardless of which
  durability policy the writer used.
- **Group-commit / batching of appends** into fewer physical writes: a policy/
  performance concern about how frames are assembled and flushed together, not a
  format concern — the resulting bytes on disk are indistinguishable from records
  appended one at a time.
- **Whether/how sealed segments are uploaded to cloud storage, and on what schedule**
  (synchronous vs. background upload) — genuinely policy. The *shape* of what gets
  uploaded (segment file bytes, unchanged) and the catalog schema (§1.2.1) is in
  scope; the upload triggering/scheduling/retry policy is not.
- **Rotation triggers** (segment size threshold, time-based rotation, etc.): policy.
  The *naming and immutability contract* of a rotated segment (§1.1) is in scope; the
  trigger that causes rotation to happen is not.
- **Compression algorithm selection heuristics** (e.g. size threshold above which a
  writer chooses to compress a value): policy/writer heuristic. The wire
  representation of *the fact that a value is compressed and by which algorithm ID*
  (tag 9, §4.2) is in scope, because a reader must be able to decode it regardless of
  which writer heuristic produced it.
- **`ReplayPolicy` (Strict vs. SalvageValidPrefix, §6.2)**: this is reader-side
  policy about what to do when corruption *is* detected — it doesn't change what
  counts as corruption, only what the reader does in response. In scope: the
  *detection* rules (§6.1–§6.3). Out of scope: which of the two responses a given
  deployment chooses.

Ambiguous / needs a decision when writing the canonical spec:

- **Confirmed, and revised** — The legacy split-marker transaction encoding (§5.3a)
  is not a deprecated/read-only compatibility path: the Rust reference
  implementation actively writes it today, as the fallback for transactions too
  large to encode as a single atomic `TxnBatch` frame. A conforming writer that
  wants interop with data produced by that implementation must be able to *emit*
  split-marker transactions, not merely decode them.
- **Resolved** — The minimum-value-size compression threshold (256 bytes, §4.3) is
  confirmed purely an internal writer heuristic with no reader-visible consequence
  — a reader must accept compressed or uncompressed values of any size regardless.
  Kept as the note in §4.3 and in `value-encoding.md`, not elevated to a normative
  invariant here, per this entry's own original recommendation.
- **Confirmed** — `writer_epoch == 0` is a deliberate reserved/sentinel value
  meaning "fencing disabled" in the Rust reference implementation (see §6.4 point
  1), not an artifact of default-value handling. Recommend the canonical spec adopt
  this as a normative reserved value rather than leaving it ambiguous.
- **TODO: verify** — The frame-header-has-no-magic design (§3.1) as a permanent
  decision vs. an area open for the shared spec to change, given both known
  implementations agree today but a real forward-incompatible payload-version bump
  has apparently never been exercised in practice.
- **Requirement, confirmed as a gap in the Rust reference implementation** — the
  format's fencing guarantee (§6.4) only holds if every WAL record a given
  `writer_epoch` ever produces genuinely postdates, in ordering terms, every record
  already durable under any lower epoch. This document's source, `lease.md` §4 step
  4, is where that invariant is actually enforced: a lease acquisition's newly
  granted epoch must be at least `max_epoch_seen` from this engine's own WAL/
  manifest recovery pass, not merely `1 +` whatever the lease file happened to
  record. `writer_epoch` is then seeded verbatim from that granted epoch at engine
  startup and never reassigned afterward — so if the lease-acquisition step doesn't
  enforce the invariant, nothing downstream will catch a violation.

  Confirmed against both reference implementations: the C# engine actually
  implements this (`MidgeFileLease.Acquire` takes a `minimumEpoch` parameter,
  threaded from a public `minimumWriterEpoch` argument on its top-level
  `Open(...)` API), settling that the mechanism is a real, deliberate,
  cross-implementation design element and not a spec artifact. The Rust engine
  does **not** implement it: its lease-acquisition API takes no equivalent
  parameter at all, WAL replay's `max_epoch_seen` (point 4 above) is computed but
  never fed back into (or even reachable from) lease acquisition, and lease
  acquisition completes in full *before* WAL replay runs at all. This is a real
  gap to close in the Rust reference implementation, not a reason to relax the
  requirement: see `lease.md` §4 step 4 for the recommended fix (add the
  equivalent parameter to midge's own API surface, giving a caller with
  out-of-band epoch knowledge the same capability the C# engine's callers already
  have) and for the caveat that even the C# engine's default (`0`) leaves this
  unprotected unless some caller actually supplies a non-default value.

  In a cloud deployment, the same `lease.epoch()` value also seeds the WAL
  catalog's `fencing_epoch` (§1.2/§1.2.1) — so lease `epoch`, local `writer_epoch`,
  and cloud `fencing_epoch` are all one number by construction, and the gap above
  applies to all three simultaneously.
