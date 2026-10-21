# Write-Ahead Log (WAL) Format

Status: draft. Sections marked "TODO: verify" record points that are not yet settled;
they are open questions for this specification, not statements about any particular
implementation. Requirements are expressed with must / must not / required / should /
may in the sense of RFC 2119, whether or not they appear in capitals.

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

### 1.1 Local history: LEDGER and linked segments

The WAL directory is named `wal`. Its history authority is the `LEDGER` file;
directory listing order is never authoritative. The directory contains the ledger and
zero or more segment files named `{segment_id:020}.wal`, with exactly twenty
zero-padded decimal digits followed by `.wal`. Segment id 42 is
`00000000000000000042.wal`.

The ledger's members are ordered oldest first. Every member before the last is sealed;
the last member is the one open segment. A reader replays only members named by the
ledger, in that order. Unlisted files are not WAL history. Retired members are removed
only from the oldest prefix; the ledger's `anchor` retains the chain link of the last
retired segment.

#### 1.1.1 Ledger bytes and authority

`LEDGER` begins with a 12-byte header:

| Offset | Size | Value |
|---|---:|---|
| 0 | 8 | ASCII magic `MDGLEDGR` |
| 8 | 4 | Ledger format version `2`, unsigned little-endian |

The header is followed by frames. Each frame is `u32 length LE`, `u32 CRC32C LE`,
then that many JSON payload bytes. CRC32C covers the four length bytes followed by
the payload. The first frame must be a `Snapshot`; later frames must be `Edit`
records that apply to the ledger state. The JSON values use externally tagged enum
variants: `{"Snapshot": <ledger>}` and `{"Edit": <ledger-edit>}`. A frame that
cannot be decoded or an edit that cannot apply is corruption, except a damaged final
frame with no later valid frame, which is a torn tail.

The snapshot records the database holder UUID, chain anchor, ordered members,
`next_segment_id`, and `seq_reserved_through`. A member records its segment header
facts, optional sealed footer facts, and optional `durable_bytes` boundary. The last
member must be open; every earlier member must be sealed. The member chain, segment
ids, sequence ranges, and next id must form a reachable ledger state.

Ledger edits are externally tagged JSON variants: `Rotate`, `Reopen`,
`AdvanceLogStart`, `SetAside`, `Rebase`, `Reserve`, `SyncOpen`, and
`SalvageOpen`. Unknown variants are rejected. A successful edit is appended and
synced before the ledger reports it durable. A snapshot is published through
`LEDGER.tmp`, sync, atomic rename, and directory sync; a torn final edit may be
discarded, but a damaged snapshot or an invalid edit is corruption. Ledger format
versions other than 2 are rejected; no migration is defined.

#### 1.1.2 Segment header, records, and footer

Every segment begins with a fixed 128-byte header. All integers are little-endian.

| Offset | Size | Field |
|---|---:|---|
| 0 | 8 | Magic `MWSEGHDR` |
| 8 | 4 | Header version `2` |
| 12 | 8 | Segment id |
| 20 | 8 | Writer epoch |
| 28 | 16 | Holder UUID bytes |
| 44 | 8 | First sequence allowed in this segment |
| 52 | 8 | Previous link's segment id |
| 60 | 8 | Previous link's last sequence |
| 68 | 16 | Previous link's xxh3-128 digest bytes |
| 84 | 40 | Reserved zero bytes |
| 124 | 4 | CRC32C of bytes 0 through 123 |

Record frames follow the header and use §3 framing with §4 payloads. Writer epoch is
stored once in the segment header; it is not a per-record TLV. Record sequences
strictly increase within a segment. Each segment after the first must name the prior
sealed segment's link, have a greater id, start after the predecessor's last sequence,
and have a writer epoch no lower than the predecessor's epoch. The holder UUID must
match the ledger's holder.

A sealed segment ends with one footer frame; no bytes may follow it. Its payload is
`MF`, footer version `1`, five little-endian u64 values
(`id`, `record_count`, `first_seq`, `last_seq`, `data_len`), a little-endian
u32 family count, that many `(cf_id: u32, max_seq: u64)` pairs in strictly
increasing `cf_id` order, and a 16-byte xxh3-128 digest of every segment byte before
the footer frame. The footer must agree with the header and all preceding records.
The sealed facts retained in the ledger and cloud catalog contain the footer link,
record count, data length, and per-family sequence maxima.

### 1.2 Cloud-backed WAL

Cloud deployments store sealed segment bytes under immutable, epoch-scoped keys:
`wal/epochs/{writer_epoch:020}/{segment_id:020}.wal`. The key's writer epoch and
segment id must match the segment header. An object is eligible for replay only when
it is named by the cloud publication catalog; an unlisted object is an orphan and
must not be treated as history.

The FORMAT 8 catalog body is format version 3. Its primary object key is
`wal/publication-catalog.v1.json`; its recovery mirror is
`wal/publication-catalog.v1.mirror.json`. The `.v1` path component is a stable
object name and does not state the JSON body's format version. The primary catalog is
updated with identity-conditional writes under the current fencing lease. The mirror
is kept synchronized from the confirmed primary body and may repair a missing or
structurally corrupt primary only when the mirror itself validates. An unsupported
catalog version or inability to validate because of a resource limit is not
repairable from the mirror. See
[`behavior/cloud-storage.md`](../behavior/cloud-storage.md) for object identity,
conditional mutation, and recovery rules.

Catalog mutations require the caller's writer epoch to equal the catalog's
`fencing_epoch`. Published segments are an ordered chain. Retirement may remove only
a sealed oldest prefix and advances `anchor` to the last retired link. The
`retired_by` field retains the shared metadata generation that authorized that
retirement, including after every catalog entry is removed. Salvage raises
`seq_floor` so a sequence in discarded history cannot be assigned again.

#### 1.2.1 Publication catalog JSON

The catalog is compact JSON with this logical shape (digest fields are arrays of
exactly 16 unsigned bytes):

| Field | Shape | Meaning |
|---|---|---|
| `format_version` | u32, exactly 3 | Catalog structure version. |
| `fencing_epoch` | nonzero u64 | Current writer epoch allowed to mutate the catalog. |
| `anchor` | `ChainLink` | Link of the last retired segment, or the genesis link. |
| `next_segment_id` | u64 | First id greater than every allocated/published segment id. |
| `seq_floor` | u64 | Sequences at or below this may have been set aside and must not be reused. |
| `retired_by` | u64 | Shared metadata generation that authorized the retired prefix; zero means none. |
| `segments` | array of entries | Published sealed segments, oldest first. |

Each segment entry has `epoch: u64`, `prev: ChainLink`, `facts: SealedFacts`,
`size_bytes: u64`, and `crc32c: u32`. `SealedFacts` contains `link`,
`record_count`, `data_len`, and `max_seq_per_cf`, an array of
`[cf_id, max_sequence]` pairs. The object key is derived from `epoch` and
`facts.link.id`; it is not stored in the catalog.

A valid catalog must satisfy all of these conditions:

- `format_version` is 3 and `fencing_epoch` is nonzero.
- `anchor` is the genesis link exactly when `retired_by` is zero; a non-genesis
  anchor requires a nonzero shared metadata generation.
- Each entry's `prev` equals the preceding link, beginning with `anchor`. Segment
  ids and last sequences strictly increase, entry epochs do not decrease, and no
  entry epoch exceeds `fencing_epoch`.
- Each entry has a nonempty sealed segment: `record_count > 0`, at least one
  `max_seq_per_cf` row, `data_len >= 128`, and `data_len < size_bytes`.
  Family ids in `max_seq_per_cf` strictly increase; every maximum is nonzero and no
  greater than the link's `last_seq`; the greatest family maximum equals
  `last_seq`; and `record_count` is at least the number of family rows.
- `next_segment_id` is greater than the head id (the last entry id, or the anchor
  id when the catalog has no entries).

A reader must verify a downloaded object against its catalog size and CRC32C, then
validate the segment header, footer, and chain identity. The catalog's list order and
links, not cloud listing order, define remote replay order. A retirement must name the
unchanged oldest prefix it observed and may not regress `retired_by`. Cross-field
chain, fencing, and shared-generation checks are not fully expressible in JSON Schema
and must also be enforced by the reader.

## 2. Terminology

| Term | Meaning |
|---|---|
| Frame | The physical unit inside a segment: an 8-byte length/CRC32C header followed by a variable-length payload. |
| Payload | The bytes carried by a record or footer frame. |
| Record | One decoded logical WAL entry: an operation or transaction marker. |
| Sequence number (`seq`) | A u64 ordering WAL records across the linked segment history. The ledger reserves sequence space and segment links enforce increasing history. |
| Writer epoch | The fencing epoch stored in each segment header. It is not repeated in record payloads. |
| Segment | A ledger-named WAL file. The final ledger member is open; preceding members are sealed. |
| Chain link | A segment id, last sequence, and xxh3-128 digest that bind the next segment to its predecessor. |
| Ledger | The `LEDGER` snapshot and edit journal that authoritatively names, orders, and retires local WAL segments. |

## 3. Frame format (physical framing layer)

Every record or segment footer is written as a length-prefixed, checksummed frame
inside a segment:

```
+-------------------+--------------+-----------------------------+
| payload_len (u32) | crc32c (u32) | payload (payload_len bytes) |
| offset 0..4       | offset 4..8  | offset 8..(8+payload_len)   |
+-------------------+--------------+-----------------------------+
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
  common to length-prefixed logs, and this format accepts it: there is no header
  CRC and no magic number in the frame header (see §3.1).

### 3.1 No magic number or version at the frame level

The 8-byte frame header carries no magic bytes and no version tag of its own —
versioning lives one layer up, in the payload encoding (§4). A frame is structurally
valid whenever `payload_len <= 64 MiB` and the trailing bytes checksum correctly; the
frame layer alone cannot distinguish a valid frame of an unknown future format from
corrupt bytes. Recovery that searches an ambiguous suffix must identify a complete,
checksummed frame whose payload decodes as a record or footer (§6.2).

**TODO: verify** whether a future revision should add a frame-level magic/version to
allow multiplexing incompatible payload encodings within one physical frame stream,
or whether the payload-level magic is sufficient permanently. FORMAT 8 uses the
payload version for this gate.

### 3.2 Zero-length payloads

`payload_len = 0` is representable by the frame format (an 8-byte frame with an empty
payload, whose CRC32C is the CRC32C of zero bytes). Whether any current payload
encoding ever legitimately produces a zero-length payload is a payload-layer question,
not a framing one — see §4.

## 4. Payload encoding (record layer)

### 4.1 Single-record payload: magic + version + TLV

Every frame payload — including that of an atomic transaction batch, whose nested
structure (§5.4) is carried *inside* this envelope as a TLV value — is encoded as:

```
+---------+---------+-------------------------------+
| magic   | version | TLV-encoded fields (repeated) |
| 2 bytes | 1 byte  |                               |
+---------+---------+-------------------------------+
```

| Field | Size | Value |
|---|---|---|
| `magic` | 2 bytes | ASCII `"MW"` (`0x4D 0x57`) |
| `version` | 1 byte | `2` (current) |

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
- **Duplicate tags are accepted; the last occurrence wins.**
- A decoder must reject the whole payload as corrupt if:
  - it is shorter than the 3-byte prefix,
  - `magic` doesn't match `"MW"`,
  - `version` doesn't match a value the decoder supports (only `2` is currently
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
| 5 | `VALUE` | no | variable | Value bytes for a value-write operation, or the nested batch payload of a `TxnBatch` record (§5.4). Presence (even of a zero-length value) is semantically distinct from absence — see §5.1. When compressed (see tag 9), holds the compressed bytes. |
| 6 | `EXPIRATION` | no | 8 bytes, u64 LE | Optional TTL/expiration timestamp, Unix epoch milliseconds. |
| 7 | `RANGE_END` | no | variable | Exclusive end key for a range-delete operation. |
| 8 | `TXN_ID` | no | 8 bytes, u64 LE | Transaction identifier, present when the record is part of a transaction. |
| 9 | `COMPRESSION` | no | 1 byte | Compression algorithm applied to `VALUE` on `Put`, `Insert`, or `TxnBatch`. On `TxnBatch`, it covers the complete nested-batch payload before decoding; nested operation values remain raw. Algorithm ID meanings are defined once, shared with other formats (e.g. SST blocks), in [`value-encoding.md`](value-encoding.md). |

All multi-byte integers in TLV values are **little-endian**.

`OP`, `CF_ID`, `SEQ`, and `KEY` are required unconditionally on every record;
everything else is optional, and its presence or absence carries meaning per §5.
The segment header supplies the writer epoch for every record in that segment.

### 4.3 Encoding notes (non-normative but relevant to compatibility)

- A writer that has a value but wants to compress it may replace the raw value bytes
  with a compressed form under tag 5 and record the algorithm under tag 9. This
  applies to the complete nested payload in a `TxnBatch` record as well. A reader
  that doesn't understand a given compression byte cannot safely decode the value —
  this makes the compression scheme part of the effective compatibility surface even
  though it's logically a value-codec concern layered on top of the WAL TLV format.
  The threshold for "worth compressing" is a minimum input size, shared with the SST
  compression path (see
  [`value-encoding.md`](value-encoding.md#1-compression-algorithm-ids)). It is a
  writer-side heuristic, not a format requirement — a reader must accept compressed
  and uncompressed values of any size.
- Presence of `VALUE` as an empty (zero-length) TLV is a legitimate, distinct wire
  state from `VALUE` being entirely absent (used to represent "Put with an empty
  string value" vs. "Delete/no value").

## 5. Record types and semantics

### 5.1 Operation kinds (`OP` tag values)

| Value | Name | Class ("role") | Semantics |
|---|---|---|---|
| 0 | `Put` | value write | Upsert `KEY` → `VALUE` at `SEQ`. `VALUE` required. |
| 1 | `Insert` | value write | Same wire/replay semantics as `Put` at the WAL layer; the distinction (e.g. insert-only conflict semantics) is an engine-level API concern, not a WAL-format concern — the WAL treats `Put` and `Insert` identically during replay. |
| 2 | `Delete` | point delete | Tombstone `KEY` at `SEQ`. `VALUE` MUST NOT be present (nor, therefore, `COMPRESSION`); a `Delete` record carrying either is corrupt. [`sst.md`](sst.md) §3.4 depends on this rule. |
| 3 | `DeleteRange` | range delete | Tombstone the half-open range `[KEY, RANGE_END)` at `SEQ`. `RANGE_END` required. |
| 4 | `TxnBegin` | transaction-begin marker | Marks the start of a (legacy-style, non-atomic-frame) transaction identified by `TXN_ID`. Carries no direct mutation. |
| 5 | `TxnCommit` | transaction-commit marker | Marks the transaction identified by `TXN_ID` as committed; on replay this is the trigger to apply all operations buffered since the matching `TxnBegin`. Carries no direct mutation of its own. |
| 6 | `TxnBatch` | atomic transaction batch | A single physical frame whose `VALUE` holds an entire transaction's operations, nested-encoded (§5.4). The outer `VALUE` may be compressed using tag 9. Carries no direct mutation itself — replay decodes and applies the nested batch. |

Every operation kind (including markers) carries `SEQ` and `CF_ID` (markers
conventionally use the default CF). Its writer epoch is inherited from the segment
header, as specified in §1.1.2.

`is_transaction_marker` = `{TxnBegin, TxnCommit, TxnBatch}`. These three carry no
direct memtable mutation on their own record; their effect is defined entirely by
§5.3–§5.4.

### 5.2 Sequence numbers and ordering

- `SEQ` is a per-database u64 that must strictly increase across the ledger's logical
  history. Each segment starts above its predecessor's last sequence. Recovery uses
  observed records and the ledger/catalog sequence reservation when assigning later
  values.
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
  transaction is valid and is applied entirely) or it does not (in which case, per
  §6, it is in the open segment's tail or is corruption).
- This is the preferred encoding for ordinary-sized transactions. Split-marker
  framing is **not** merely a read-path compatibility concern: it remains an
  actively emitted writer path for transactions too large to buffer as a single
  atomic `TxnBatch` frame — the "spill" path. A writer that emits only `TxnBatch`
  cannot represent a transaction larger than one frame, and will diverge from
  conforming writers on oversized transactions.

Nested transaction markers (a batch containing another `TxnBegin`/`TxnCommit`/
`TxnBatch`) are invalid and must be rejected.

### 5.4 Nested transaction batch payload (inside a `TxnBatch` record's `VALUE`)

The `VALUE` field of a `TxnBatch` record (tag 5, itself a TLV value inside the outer
§4.1 envelope) holds its own self-describing structure. If the outer record carries
`COMPRESSION` (tag 9), the reader decompresses the outer `VALUE` before decoding this
structure. The nested operation values below are always stored raw.

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
+--------+---------+---------+----------+----------------------------+---------------------------+-------------------------------+
| op     | cf_id   | seq     | exp_flag | [exp: u64] key (len+bytes) | value: flag + [len+bytes] | range_end: flag + [len+bytes] |
| 1 byte | 4 bytes | 8 bytes | 1 byte   |                            |                           |                               |
+--------+---------+---------+----------+----------------------------+---------------------------+-------------------------------+
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
remaining bytes *before* attempting a per-record allocation, so that a
corruption- or attacker-controlled `op_count` cannot induce a large allocation from
a small buffer.

Trailing bytes after the last nested record (i.e. input not fully consumed) is
corruption.

Cross-checks a decoder must perform against the **outer** record (in addition to the
inner-payload self-consistency above):
- outer `TXN_ID` == inner `txn_id`
- outer `SEQ` == inner `commit_seq`

Nested operation values inside the batch have no compression field of their own and
are always stored raw. `COMPRESSION` applies only to the outer `VALUE` bytes, not to
individual nested values.

## 6. Recovery semantics: ledger order, valid prefixes, and fencing

A reader reconstructs WAL history from the ledger and the cloud catalog, never by
sorting a directory or object listing. It validates every segment against the
recorded header, chain, and sealing facts before applying its records.

### 6.1 Segment order and chain validation

For local history, replay the ledger's members oldest first. For cloud history, replay
the catalog's segments oldest first, beginning at its anchor. Validate that every
segment id matches its file/key and header, each header names the previous chain link,
sequence ranges increase, and writer epochs do not regress. Sealed segments must have
a valid footer that agrees with all preceding frames and the recorded sealed facts.
Only the final ledger member is open and may lack a footer.

A frame must have a complete 8-byte header, a payload no larger than 64 MiB, and a
matching CRC32C. Record payloads must decode under the supported MW version and
operation rules. A footer is recognized by its MF magic and footer version; it is
valid only at the end of a sealed segment.

### 6.2 Torn tails and corruption

A partial or damaged final frame may be tolerated only in the ledger's open segment,
and only when the persisted fsync boundary proves that the verified prefix contains
all bytes acknowledged as durable. A sealed segment must verify in full; it cannot
have a tolerated torn tail. A segment shorter than its recorded durable boundary is
corruption.

Strict recovery fails on corruption. An explicit salvage mode may retain the verified
prefix of the open segment, record the corruption, cut the segment at that prefix,
and continue from a new segment. Salvage must preserve the greatest sequence that
could have been present in discarded verified data so those values cannot be assigned
again. Corruption inside a sealed segment or in the ledger is not an ordinary torn
tail.

If a failed frame hides later independently verifiable frames, recovery must not treat
those later records as part of the verified prefix. Salvage may discard them only
while retaining their highest observed sequence as a no-reuse floor. A reader must
never silently reinterpret a damaged length as a valid boundary that loses later
acknowledged data.

### 6.3 Sequence reservation and salvage floor

The ledger's `seq_reserved_through` records sequence numbers already handed out or
reserved. A `Reserve` edit must raise this value; it must not move backward.
Sequence gaps are permitted. A later writer starts above the maximum of the durable
ledger reservation, recovered records, and any salvage floor.

The cloud catalog's `seq_floor` carries sequence numbers that may have belonged to
WAL discarded during salvage. It is monotonic and prevents those sequence numbers
from being assigned again after local recovery or cache replacement.

### 6.4 Writer-epoch fencing

The writer epoch is present in each segment header, not in individual record payloads.
Every record in one segment therefore has the same epoch. A segment after another
segment may retain the same epoch or move to a higher one; it must never lower the
epoch. Local ledger facts and cloud catalog entries must agree with the segment's
header epoch.

A writer acquiring a lease must choose an epoch above every epoch that could already
be durable in the database's WAL, as required by `lease.md` §4. A stale writer must
fail its lease-fenced mutations. Recovery does not filter individual records by epoch:
the lease and monotonic segment-chain rules prevent a superseded writer from
publishing authoritative later history. The catalog's `fencing_epoch` must match
the epoch authorized to mutate it.

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

Open questions:

- **TODO: verify** — Whether the frame header's lack of a magic/version (§3.1) is a
  permanent decision or open for revision. No forward-incompatible payload-version
  bump has ever been exercised against it.

Cross-document dependency: lease acquisition (`lease.md` §4) must choose an epoch
above every writer epoch that could already be durable in this database. The granted
lease epoch is stored in each new segment header and is also the cloud catalog's
`fencing_epoch` (§1.2, §1.2.1). A segment contains records from exactly one writer
epoch; epochs may rise along the segment chain but may not regress.
