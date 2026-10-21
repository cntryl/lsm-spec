# Manifest Format

Status: draft. Sections marked "TODO: verify" record points that are not yet settled;
they are open questions for this specification, not statements about any particular
implementation. Requirements are expressed with must / must not / required / should /
may in the sense of RFC 2119, whether or not they appear in capitals.

## 1. Overview

The manifest is the source of truth for an LSM-tree storage engine's **file-level
structure**: which SST files currently exist and are authoritative for reads, which
column families (independent keyspaces) exist, and the per-family flush frontier and
SST sequence counters that must survive a restart. WAL sequence allocation and
retirement live in the WAL ledger; cloud WAL retirement authority lives in the
publication catalog and shared metadata generation. The manifest answers "what does
the tree look like right now," as opposed to the WAL (what mutations have been made
durable) or an SST file (what a specific sorted run contains).

Structurally the manifest is **not** one file but a small persistence scheme built
from three pieces:

1. **A snapshot** (`manifest.snapshot.json`) — a full, self-contained serialization of
   manifest state as of some point, tagged with a durable **edit checkpoint id**
   marking how much journal history it already reflects.
2. **A journal** (`manifest.journal`) — an append-only, checksummed log of incremental
   **edits** (add/remove a file, create/drop a column family, bump a counter, etc.)
   applied on top of the snapshot. This is conceptually a mini write-ahead log
   dedicated to manifest state, with its own framing/checksum scheme independent of
   the data WAL described in [`wal.md`](wal.md).
3. **A legacy base file** (`manifest.json`) — an older full-state file a reader may
   use only when no snapshot exists. A FORMAT 8 writer does not create or update this
   file; when a snapshot exists it is ignored on load.

A reader reconstructs current state by loading the most recent snapshot (or the
legacy base file, or an empty default manifest, depending on what exists — see §5.1),
then replaying journal edits recorded after that snapshot's checkpoint, in order
(§4).

### 1.1 File naming and lifecycle

Manifest state lives at fixed, non-rotating filenames directly under the database
root (not a subdirectory, unlike the WAL directory):

| File | Role |
|---|---|
| `FORMAT` | On-disk format version marker for the whole database directory (not manifest-specific — see §2). |
| `manifest.snapshot.json` | Authoritative full-state snapshot, when present. |
| `manifest.snapshot.json.tmp` | Staging file for an in-progress snapshot write (see §6.3). |
| `manifest.journal` | Append-only log of edits since the last snapshot checkpoint. |
| `manifest.journal.repair.tmp` | Staging file used only when truncating a torn journal tail (§6.2). |
| `manifest.json` | Legacy full-state base accepted only when no snapshot exists; FORMAT 8 writers do not write it. |

None of these files are content-addressed or rotated by an incrementing id the way
WAL segments are (§1.1 of `wal.md`) — the manifest instead uses a **snapshot +
journal-since-checkpoint** scheme, which serves an analogous "don't replay unbounded
history" purpose (compaction of the journal, §7) without file rotation.

The combination "no snapshot, but a journal is present" is valid. In that case a
reader uses `manifest.json` as the base when it exists, or the default manifest when
it does not, then replays the durable journal suffix (§5.1–§5.2).

## 2. Format version marker

A single-line text file named `FORMAT` at the database root gates whether the
directory's on-disk layout (manifest scheme, WAL scheme, SST scheme — the whole
on-disk contract, not just the manifest) is understood by the opening engine:

```
midge-format-version=<version>\n
```

- `midge-format-version` is a fixed literal key. It must be written and matched
  verbatim; the name carries no meaning beyond identifying this marker.
- A conforming writer emits the canonical decimal value **8**. A reader accepts a
  marker only when the value parses as an unsigned decimal integer equal to 8; leading
  or trailing whitespace, a sign, non-ASCII digits, or a missing final newline are
  invalid. Leading zeroes are accepted by the reader but must not be emitted by a
  writer.
- The marker is written once, atomically (temp file + durable rename), the first
  time a fresh database directory is opened. It is never rewritten in place after
  that.
- Opening a directory that already has persisted state (a manifest file, a manifest
  snapshot, a journal, or non-empty `wal`/`sst` subdirectories) but **no** `FORMAT`
  marker is refused — such state is treated as pre-versioning legacy data that must
  be rebuilt/re-imported, not silently accepted.
- Opening a directory whose marker names any version other than the current
  understood value is refused (both "older, no longer supported" and "newer, not yet
  understood" are rejected identically at this layer — there is no defined
  migration/negotiation path in-format).

This marker is **not** manifest-specific — it is the format gate for the whole
database directory: opening a directory whose `FORMAT` value the engine doesn't
understand is refused outright, regardless of which structure (WAL, SST, or
manifest) within it actually changed between versions. A reader supporting this
profile supports only value 8; it must reject older and newer values without
rewriting them. No in-place migration is defined.

This integer is not the same value as, and has no arithmetic relationship to, the
version fields embedded in individual structures. The FORMAT 8 profile uses SST
footer version 4 (`sst.md` §6–§7), WAL record payload version 2 (`wal.md` §4.1), WAL
ledger version 2, WAL segment header version 2 and footer version 1, and cloud WAL
publication catalog version 3 (`wal.md` §1.2.1). Each gate is checked independently;
none is inferred from another. A reader must reject any structure version it does not
support even when the directory marker is 8.

The FORMAT marker is created only after the root has been established as a new
database. A root containing a manifest, journal, intent log, or non-empty `wal` or
`sst` directory without `FORMAT` is persisted state and must be rejected. Unrelated
root entries such as operating-system metadata files do not, by themselves, make a
root persisted state.

## 3. Terminology

| Term | Meaning |
|---|---|
| Manifest | The full logical state: file set, column-family set, and durability counters, as reconstructed by loading a snapshot (or legacy base file, or default) and replaying journal edits on top. |
| Snapshot | A complete, self-contained serialization of manifest state at some point, tagged with an edit checkpoint id. |
| Journal | The append-only, checksummed log of edits recorded since the referenced snapshot's checkpoint. |
| Edit / manifest edit | One logical, atomic change to manifest state (e.g. "add this SST", "create this column family"). The unit of journal replay. |
| Edit id | A durable, monotonically increasing u64 identity stamped on every appended edit (or edit batch), used to determine which edits a given snapshot already reflects. |
| Edit checkpoint id | The highest edit id already folded into a given snapshot. Journal replay against that snapshot need only consider edits with a strictly higher id. |
| Fsync marker | A special journal record type that marks the point up to which appended edits are considered durable; edits after the last marker in a file are not replayed (§4.4). |
| Column family (CF) | An independent, named keyspace with its own id, lifecycle, and set of owned SST files. |
| File metadata (`FileMeta`) | The manifest's record describing one SST file: name, level, size, owning column family, key/sequence bounds, checksum. |

## 4. Journal: framing, record types, and checksums

### 4.1 Record framing

Every journal record (an edit, a batch of edits, or an fsync marker) is written as a
length-prefixed, checksummed record — structurally similar to, but a distinct scheme
from, the WAL's frame format (`wal.md` §3):

```
+------------------+-------------------+-----------------------------+------------------+
| record_type (u8) | payload_len (u32) | payload (payload_len bytes) | crc32 (u32)      |
| offset 0..1      | offset 1..5       | offset 5..(5+payload_len)   | trailing 4 bytes |
+------------------+-------------------+-----------------------------+------------------+
```

| Field | Size | Byte order | Description |
|---|---|---|---|
| `record_type` | 1 byte | — | Discriminates the payload's logical type; see §4.2. Also used as a structural sanity check: a deserialized edit's own reported type (from its enum tag) must match this byte, or the record is corrupt. |
| `payload_len` | 4 bytes | little-endian | Length of the following payload. |
| `payload` | `payload_len` bytes | — | JSON-serialized edit envelope (or fsync marker) — **not** a fixed binary layout; see §4.3. |
| `crc32` | 4 bytes | little-endian | CRC-32 (**not** CRC32C — see note below) of the payload bytes only. |

Note the checksum algorithm difference from the WAL: the manifest journal uses plain
CRC-32 (the `crc32fast` crate's default, i.e. the classic zlib/gzip/IEEE polynomial),
while the WAL frame format uses CRC32C (Castagnoli). This is a deliberate
per-structure difference, not an error in this document: a conforming
implementation must use CRC-32 here even though it uses CRC32C for the WAL
(`wal.md` §3) and for SST blocks (`sst.md` §5.2).

`payload_len` is an unsigned 32-bit value, so one record payload is at most
`u32::MAX` bytes. The format defines no smaller payload limit.

### 4.2 Record type registry

The FORMAT 8 journal uses these record types:

| Value | Meaning |
|---|---|
| 1 | `AddSst` |
| 2 | `RemoveSst` |
| 3 | `CreateColumnFamily` |
| 6 | `BumpNextSstSeq` |
| 8 | `Batch` |
| 9 | Fsync marker (not an edit; see §4.4) |
| 10 | `DropColumnFamilyAt` |
| 11 | `ReclaimColumnFamily` |
| 12 | `AdvanceFlushFrontier` |

Values 4, 5, and 7 are unsupported in this profile. Any other record type is
corruption; journal records are not skippable. The `record_type` byte must agree
with the externally tagged edit variant in the payload. A FORMAT 8 reader must not
interpret records from an older journal profile as FORMAT 8 edits.

### 4.3 Payload encoding: edit envelopes

An edit or batch payload is a JSON object containing a durable edit id and one
externally tagged edit:

```jsonc
{
  "edit_id": <u64>,
  "edit": { "VariantName": <fields> }
}
```

The `edit_id` is a monotonically increasing u64 identity. The `edit` union uses
variant names as JSON keys. The newtype `AddSst` variant inlines its `FileMeta` object;
`Batch` contains an ordered array of recursively tagged edits. Every FORMAT 8 journal
edit must use this envelope; a bare legacy enum without `edit_id` is invalid.

The FORMAT 8 edit variants are:

| Variant | Fields | Semantics |
|---|---|---|
| `AddSst` | one `FileMeta` (§5.5) | Add one SST identity. Repeating the same identity is idempotent; a conflicting identity for the same name is corruption. |
| `RemoveSst` | `name: string` | Remove the named SST if present. |
| `CreateColumnFamily` | `id: u32`, `name: string`, `created_at: u64` | Create a non-default family. A repeated original create is idempotent; ids are not reused. |
| `DropColumnFamilyAt` | `id: u32`, `drop_sequence: u64`, `dropped_sst_names: [string]` | Tombstone a family and capture its snapshot frontier and owned SST set. An empty name list is normalized from the family's current files when applied. |
| `ReclaimColumnFamily` | `id: u32`, `names: [string]` | Remove captured SST names from a dropped family; retain its tombstone. Reclamation can proceed in batches. |
| `BumpNextSstSeq` | `cf_id: u32`, `next_seq: u64` | Set the per-family allocator to `max(current, next_seq)`. |
| `AdvanceFlushFrontier` | `cf_id: u32`, `freeze_sequence: u64` | Advance one live family's persisted flush authority as part of the SST publication batch; see §5.7. |
| `Batch` | `[ManifestEdit]` | Validate and apply the ordered child edits atomically as one journal record. |

SST names in `AddSst`, `RemoveSst`, `DropColumnFamilyAt.dropped_sst_names`, and
`ReclaimColumnFamily.names` must be valid persisted SST names: a single relative
path component with no `/`, `\`, `:`, or NUL, ending in `.sst`. Invalid names are
corruption. A batch must validate every child before changing manifest state.

Applying edits is replay-safe where the transition is defined as idempotent: an
identical `AddSst`, the original `CreateColumnFamily`, a repeated drop at the same
frontier, and repeated monotonic counter/frontier values do not regress state.
Conflicting identities, family id reuse, or regressing frontiers are rejected.

### 4.4 Fsync markers and the durability boundary

Record type 9 is a JSON `FsyncMarker`, not an edit:

```jsonc
{ "last_persisted_sequence": <u64>, "ts_millis": <u64> }
```

`last_persisted_sequence` stores the edit id of the immediately preceding edit or
batch record (the field name is retained for compatibility); `ts_millis` is an
unsigned Unix-millisecond timestamp. Each append writes one edit or batch followed
by one marker and syncs both before reporting success. Replay applies only edits
followed by a valid marker. A complete edit at EOF without its marker is not durable.

## 5. Recovery: loading a manifest

### 5.1 Base-state precedence

On load, exactly one of the following supplies the starting (pre-journal-replay)
manifest state, in this precedence order:

1. **`manifest.snapshot.json`**, if it exists — parsed as the full JSON `Manifest`
   structure (§5.5/§5.6 below). This is authoritative whenever present; step 2 is not
   consulted at all if this file exists, even if it disagrees with `manifest.json`.
2. **`manifest.json`** (the legacy base), if no snapshot exists but this does.
3. An empty **default** `Manifest` if neither file exists. It has the required live
   default-family frontier `{ "cf_id": 0, "through_sequence": 0 }`, no files or
   non-default column families, an empty `next_sst_seqs` map, and
   `edit_checkpoint_id = 0`. WAL sequence state is read from `wal/LEDGER`, not from
   the manifest.

`manifest.snapshot.json.tmp` is never an authoritative source at load time. Only the
final snapshot name is read, so a crash mid-write leaves the previous durable state
intact and the stray temporary file inert until a later write or cleanup.

### 5.2 Journal replay on top of the base state

If `manifest.journal` exists, a reader replays every valid edit recorded in it whose
**edit id is strictly greater than** the base manifest's `edit_checkpoint_id`,
applying each with `apply_edit` (§4.3) in the order the journal records them. Edits
at or below the checkpoint id are skipped — they are already folded into the base
state, and (per §4.3's idempotency requirement) it would be harmless but wasteful to
re-apply them.

This checkpoint-gated replay — rather than "replay the whole journal every time," or
"trust that the journal was always truncated exactly at snapshot time" — is what
makes recovery correct across the specific crash window between "snapshot file
durably renamed into place" and "journal file durably truncated" (§6.3's
snapshotting process): if a crash lands in that window, the journal is untruncated
and still contains edits already reflected in the new snapshot, but replaying only
edits above the recorded `edit_checkpoint_id` re-derives the same state without
double-applying them.

If journal replay itself fails (a real, unrecoverable corruption per §6.1, as opposed
to a tolerated partial tail), the outcome depends on a caller-supplied recovery
policy: **Strict** propagates the failure and the load fails outright; a permissive
policy logs a warning and proceeds with the base state alone, silently forgoing any
edits recorded in the journal since the checkpoint.

After base-state loading and journal replay, every `FileMeta.name` and each SST name
retained by a column-family tombstone is validated as a well-formed persisted SST name
(§4.3); a validation failure at this point fails the load, even if the underlying
JSON/journal parsing succeeded.

### 5.3 Column-family lifecycle

`column_families` contains records for non-default families. The default family has
id `0`, is always live, and has a frontier row, but has no `ColumnFamilyMeta` entry.
The default family cannot be created or dropped through journal edits.

A non-default family moves through these states:

1. **Active**: `deleted_at` is absent. `CreateColumnFamily` requires an id greater
   than zero, a name other than `default`, and a name not used by another live family.
   A repeated create is valid only when its id, name, and `created_at` match the
   original create.
2. **Tombstoned**: `DropColumnFamilyAt` records `deleted_at`, `drop_sequence`, and
   the SST names owned by the family. If its `dropped_sst_names` is empty, applying
   the edit captures the family's current SST names. The family's frontier becomes
   the reserved `u64::MAX` dropped sentinel. The id remains permanently allocated.
3. **Reclaimed**: `ReclaimColumnFamily` removes captured SST names from the manifest
   and removes those names from the tombstone. When the captured list is empty,
   `reclaimed` becomes true. The tombstone and its id remain in the manifest.

A `CreateColumnFamily` edit cannot revive a tombstoned family or reuse its id, even
if its SSTs were reclaimed. Replaying the original create after a later drop is a
no-op; it must not clear the tombstone. The layer deciding when no live snapshot can
observe a dropped family's data may choose when to append reclamation edits; the
journal contract requires only the stated tombstone and reclamation transitions.

### 5.4 File-set (version-edit) semantics

The manifest's file list (`files: [FileMeta]`) is flat; `cf_id` and `level` are
fields on each entry. A reader reconstructs files for a family and level by filtering
this list. `AddSst` adds a new name or accepts a repeat only when the existing SST
identity matches; the same name with conflicting persisted identity is corruption.
`RemoveSst` removes the named entry if present. A batch validates all child edits
against staged state before committing any of them.

There is no explicit "version" object distinct from the manifest's current file
list — unlike some other LSM manifest designs (e.g. LevelDB/RocksDB's `Version` +
`VersionEdit` model with an explicit level-sorted-run structure), this format's
"current version" is simply "the file list as of the most recently applied edit,"
recomputed by linear replay rather than through an explicit persisted version-number
object. Implementations may keep additional in-memory read structures, but they do not
change this persisted flat file-set contract.

### 5.5 File metadata fields

Each `FileMeta` entry:

| Field | Type | Required in JSON? | Semantics |
|---|---|---|---|
| `name` | string | yes | Persisted SST filename; see §4.3's validation rule. Writers commonly use `{cf_id:06}_{level:02}_{sst_seq:020}.sst`, but the manifest contract requires only one safe relative `.sst` filename; the padded naming pattern is not required for reading or identity. |
| `level` | u32 | yes | LSM level the file belongs to. |
| `size_bytes` | u64 | yes | File size in bytes. |
| `content_crc32c` | u32, optional | no | Whole-file CRC32C, when recorded, used to validate the SST's bytes independent of the manifest journal's own CRC-32 (§4.1) — this checksum is over the *referenced SST file's content*, a different algorithm and a different object than the journal record checksum. |
| `cf_id` | u32 | yes in canonical output; readers default omission to 0 | Owning column-family id. |
| `sst_seq` | u64 | yes in canonical output; readers default omission to 0 | Per-family sequence number identifying this file's creation order; related to the `next_sst_seqs` counter advanced by `BumpNextSstSeq`. |
| `smallest_key` / `largest_key` | nullable string | yes in canonical output; readers default omission to null | Inclusive key bounds covered by the file, when known. A present byte string is lowercase hexadecimal with two characters per byte. A missing or null bound is unknown. |
| `smallest_seq` / `largest_seq` | nullable u64 | yes in canonical output; readers default omission to null | Inclusive sequence-number bounds of entries in the file, when known. |
| `key_bounds_complete` | boolean | yes in canonical output; readers default omission to `false` | Whether the recorded key bounds include every point key and every range-tombstone endpoint in the SST. A reader must treat a file with `false`, a missing field, or incomplete bounds as a candidate for every key-range lookup; it must not use those bounds to prune the file. |
| `sublevel` | u32 | yes in canonical output; readers default omission to 0 | Sub-ordering within `level`; lower values sort first within the level. |

A read-access counter (`read_count`) exists on the in-memory structure for
compaction-prioritization heuristics but is explicitly excluded from serialization
(`#[serde(skip)]`) — it is pure runtime state, never persisted, and out of scope for
this format document.

### 5.6 Manifest-level (top-level) fields

| Field | Type | Requirement | Semantics |
|---|---|---|---|
| `flushed_through` | array of `{cf_id, through_sequence}` | Required | Flush authority for every family. Rows are strictly ordered by `cf_id`, begin with the live default family `0`, and contain exactly one row per family. See §5.7. |
| `files` | `[FileMeta]` | Required in canonical output; readers default omission to `[]` | The flat file set; see §5.4. |
| `column_families` | `[ColumnFamilyMeta]` | Required in canonical output; readers default omission to `[]` | All non-default column-family records, including tombstones; see §5.3. |
| `next_sst_seqs` | map `u32 -> u64` | Required in canonical output; readers default omission to `{}` | Per-family next-SST-sequence counters keyed by canonical unpadded decimal `cf_id`. A missing key allocates sequence `0`. |
| `edit_checkpoint_id` | u64 | Required in canonical output; readers default omission to `0` | The journal replay horizon this snapshot already reflects; see §5.2. |

The whole `Manifest` structure is serialized as JSON (not a binary format) in the
snapshot and in a legacy base file when one is being read. FORMAT 8 writers persist
the full state in the snapshot, not in `manifest.json`. The WAL sequence reservation
and history boundary are persisted in `wal/LEDGER` (`wal.md` §1.1). Cloud retirement
generations are persisted in the cloud WAL publication catalog (`wal.md` §1.2.1).

### 5.7 Per-family flush frontiers

Each `flushed_through` row contains exactly two unsigned integer fields:
`cf_id` and `through_sequence`. Rows must be strictly ordered by increasing `cf_id`
with no duplicates. The first row must be the live default family (`cf_id = 0`), and
its `through_sequence` must not equal `u64::MAX`. Every non-default family in
`column_families` must have exactly one frontier row, and every non-default frontier
row must name a family in `column_families`.

For a live family, `through_sequence` is the highest sequence for which the
manifest's published SST set is authoritative. `u64::MAX` is reserved as the dropped
family sentinel: a family row with this value must have a tombstone (`deleted_at` is
present), and a tombstoned family must use this value. The sentinel is not a valid
flush sequence for a live family.

`AdvanceFlushFrontier` is valid only inside a `Batch`. Before the frontier edit, the
same batch must add at least one new SST for that family, each with a known
`largest_seq` no greater than `freeze_sequence`. The batch must keep those SSTs,
advance a family at most once, and must not advance a dropped family, use the reserved
`u64::MAX`, or regress its prior frontier. A frontier greater than the prior value
cannot be established by reusing an existing SST.

Frontiers are durable authority, not a hint. A manifest edit must not remove a
family's frontier, lower a live frontier, or change a dropped frontier back to a live
value. A reader must reject a missing, duplicate, out-of-order, orphaned, or
lifecycle-inconsistent frontier rather than inventing a default value. When publishing
a newer shared metadata generation, the new frontier set must retain every prior row,
never lower a live sequence, and never revive a dropped family.

## 6. Recovery semantics: corruption and torn tails

### 6.1 Hard corruption in the journal

Any of the following, encountered while scanning `manifest.journal`, is corruption
and is never silently tolerated as a benign tail (contrast §6.2):

- A `record_type` byte not listed in §4.2.
- A CRC-32 mismatch between a record's declared checksum and the checksum computed
  over its actual payload bytes.
- A record whose JSON payload does not match the FORMAT 8 edit envelope or the
  fsync-marker shape (§4.3–§4.4).
- A record whose deserialized edit's own reported variant/type does not match the
  `record_type` byte that framed it.
- An SST name anywhere in a decoded edit that fails persisted-name validation
  (§4.3).
- Mid-file corruption that precedes an otherwise-valid later record: unlike the
  "verified-suffix" carve-out in the WAL format (`wal.md` §6.3) that can turn a
  would-be torn-tail into a hard error by finding recoverable data past it, here the
  relationship runs the *other* direction — the manifest journal replay simply fails
  the moment it hits a corrupt record it cannot classify as a genuine end-of-file
  condition (§6.2), even if bytes that would otherwise re-parse as valid records
  follow it. There is no scan-ahead/resynchronization mechanism analogous to the
  WAL's magic-based resync scan.

### 6.2 Tolerated incomplete tail vs. required repair

Two distinct "the journal ends abruptly" situations are handled differently:

1. **Partial record at EOF** (not enough bytes remain to read a declared header, or
   the declared `payload_len` + trailing CRC would run past the file's actual
   length): this is treated as an ordinary consequence of a crash mid-append and is
   **not** an error. Replay stops at the last position it could fully verify.
2. **A fully-formed, CRC-valid edit record present at EOF with no following fsync
   marker record** (§4.4): even though the edit record itself parses and checksums
   correctly, it is excluded from the replayed edit set, because the fsync marker is
   what asserts "this edit is durable." This is functionally equivalent to the
   partial-record case for the purposes of *what gets applied*, but is a distinct
   condition from a bytes-level "torn record."

When either condition is detected, **before the next append to the journal
succeeds**, the reader rewrites the journal file to contain only the verified-durable
prefix: it reads the journal up through the last fsync marker's recorded end offset,
writes those bytes to `manifest.journal.repair.tmp`, and atomically renames that into
place as `manifest.journal`. This repair-on-next-write behavior means a journal left
with an unmarked tail after a crash is self-healing the next time the engine opens
the database and performs a write — no separate offline repair tool is required by
the format, though the repair *is* itself part of the reader/writer contract (an
implementation must perform it to remain compatible with concurrent writers
expecting a clean tail).

### 6.3 Snapshot/journal crash-consistency window

Publishing a new snapshot is itself a multi-step process (detailed as policy in §7,
but with format-relevant consistency implications here):

1. Merge the journal's post-checkpoint edits into a full manifest state.
2. Write that merged state to `manifest.snapshot.json.tmp`, then atomically rename
   it to `manifest.snapshot.json`.
3. Truncate `manifest.journal` to empty.

A crash between steps 2 and 3 leaves a durable snapshot **and** an untruncated
journal whose already-merged edits are still physically present. This is exactly the
scenario §5.2's checkpoint-gated replay is designed to make harmless: the new
snapshot's `edit_checkpoint_id` already reflects those edits, so replay skips them
even though their bytes remain on disk; a subsequent successful checkpoint operation
truncates the journal for real. A crash before step 2's rename completes leaves the
old snapshot (or no snapshot) and the full journal intact — ordinary §5.1/§5.2
recovery applies unmodified. The checkpoint writer must preserve the journal edits represented by the snapshot
checkpoint it publishes; stale or competing writers must be fenced or serialized so a
lower checkpoint cannot discard newer durable edits.

## 7. Format vs. policy

The following are explicitly **behavioral/policy**, not part of the on-disk format
contract, and are not required for wire compatibility between two engines that both
correctly implement §2–§6:

- **When a snapshot is taken (manifest-journal "compaction" trigger)** — e.g. journal
  size threshold, edit count threshold, time-based, or "on every N flushes/
  compactions": entirely a policy/performance decision about how often to pay the
  cost of a full-state rewrite versus letting the journal grow. The format only
  specifies *what a snapshot+checkpoint operation must produce* (§6.3) and *how a
  reader must reconcile* the resulting files (§5.1–§5.2) — not when a writer chooses
  to trigger one.
- **Whether edits are appended individually or batched** (`AddSst` vs. a `Batch` of
  several edits in one journal record): both are format-legal (single-edit records
  or record type 8), and a reader must handle either. Whether a given writer chooses to batch (e.g.
  to make a flush's "add these N new SSTs" atomic as one journal record) is a
  performance/atomicity-of-intent choice by the writer, not a format requirement.
  (Note this *is* format-relevant in one sense: choosing to batch several edits
  changes what atomicity a reader observes across a crash — but the *decision* of
  when to batch is policy; the *batch record's* structure and replay-atomicity
  guarantee is format, §4.3.)
- **Fsync scheduling** beyond the required per-append durability boundary (§4.4) —
  implementations may group underlying I/O only if each successful append returns
  after its edit and marker are durable.
- **Column-family reclamation timing** (exactly when, relative to snapshot pinning
  and compaction scheduling, a `ReclaimColumnFamily` edit is actually appended once a
  dropped family's files become eligible): policy about *when* to reclaim. The
  format only specifies the states a column family passes through and what each
  transition's edit must contain (§5.3).
- **Concurrent-writer coordination beyond what correctness requires**: the choice of
  serialization mechanism (an in-process lock, a filesystem lock, a lease) is an
  implementation concern. The format's crash-consistency guarantees (§6.3) hold
  regardless, provided *some* mechanism prevents two writers from interleaving bytes
  within a single journal record append.
- **An explicit salvage operation** may preserve a verified durable prefix after
  reporting corruption; it must not classify an interior corrupt record as a normal
  torn tail or silently claim a complete recovery.

**Settled:**

- `ManifestEdit`'s tagged-union JSON shape is externally tagged
  (`{"VariantName": <fields>}`); see §4.3, which states the exact shape an
  implementation must reproduce.
- FORMAT 8 does not define the older `DropColumnFamily` journal variant. Only
  `DropColumnFamilyAt` records a family drop.
- FORMAT 8 readers may use `manifest.json` only as a base when no snapshot exists;
  FORMAT 8 writers do not write or synchronize this legacy file. See §1.1 and §5.1.

**Open questions:**

None for the manifest rules specified here.
