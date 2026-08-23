# Manifest Format

Status: draft. Sections marked "TODO: verify" record points that are not yet settled;
they are open questions for this specification, not statements about any particular
implementation. Requirements are expressed with must / must not / required / should /
may in the sense of RFC 2119, whether or not they appear in capitals.

## 1. Overview

The manifest is the source of truth for an LSM-tree storage engine's **file-level
structure**: which SST files currently exist and are authoritative for reads, which
column families (independent keyspaces) exist, and a handful of monotonic counters
(next WAL sequence, next per-column-family SST sequence, a cloud durability
checkpoint) that must survive a restart. It answers "what does the tree look like
right now," as opposed to the WAL (what mutations have been made durable) or an SST
file (what a specific sorted run contains).

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
3. **A legacy mirror** (`manifest.json`) — a plain full-state file kept in sync with
   the snapshot for backward compatibility / human debuggability. It is read only
   when no snapshot exists (see §5.1); when a snapshot is present it is ignored on
   load and is not authoritative.

A reader reconstructs current state by loading the most recent snapshot (or the
legacy mirror, or an empty default manifest, depending on what exists — see §5.1),
then replaying journal edits recorded after that snapshot's checkpoint, in order
(§4).

### 1.1 File naming and lifecycle

All manifest state lives at fixed, non-rotating filenames directly under the database
root (not a subdirectory, unlike the WAL's `wal_dir`):

| File | Role |
|---|---|
| `FORMAT` | On-disk format version marker for the whole database directory (not manifest-specific — see §2). |
| `manifest.snapshot.json` | Authoritative full-state snapshot, when present. |
| `manifest.snapshot.json.tmp` | Staging file for an in-progress snapshot write (see §6.3). |
| `manifest.journal` | Append-only log of edits since the last snapshot checkpoint. |
| `manifest.journal.repair.tmp` | Staging file used only when truncating a torn journal tail (§6.2). |
| `manifest.json` | Legacy/compatibility full-state mirror, kept in sync whenever a snapshot is saved. |
| `manifest.json.tmp` | Staging file for a `manifest.json` write. |

None of these files are content-addressed or rotated by an incrementing id the way
WAL segments are (§1.1 of `wal.md`) — the manifest instead uses a **snapshot +
journal-since-checkpoint** scheme, which serves an analogous "don't replay unbounded
history" purpose (compaction of the journal, §7) without file rotation.

**TODO: verify** whether the combination "no snapshot, but a journal is present" is
reachable at all, or whether a missing snapshot always implies pre-journal-era state
in which only `manifest.json` exists. §5.1's precedence rule defines the outcome
either way; what is unsettled is whether the legacy-mirror-plus-journal case can
occur in practice.

## 2. Format version marker

A single-line text file named `FORMAT` at the database root gates whether the
directory's on-disk layout (manifest scheme, WAL scheme, SST scheme — the whole
on-disk contract, not just the manifest) is understood by the opening engine:

```
midge-format-version=<version>\n
```

- `midge-format-version` is a fixed literal key. It must be written and matched
  verbatim; the name carries no meaning beyond identifying this marker.
- `<version>` is a decimal, unsigned integer. Current value: **3**.
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
manifest) within it actually changed between versions. It is documented here
because the manifest's own on-disk shapes (below) are only guaranteed to match this
document when `FORMAT` reads 3.

**This integer is not the same value as, and has no arithmetic relationship to, the
version fields embedded in individual structures** — e.g. the SST footer's own
`format_version` (currently **4**, see `sst.md` §6–§7) and the WAL payload
envelope's `version` byte (currently **1**, see `wal.md` §4.1). All three are
independent gates, each checked on its own. A conforming implementation must validate
every one of them where it applies, and none may be inferred or derived from another.

**TODO: verify** whether a future bump of the directory-level `FORMAT` value is meant
to imply anything about the SST or WAL sub-format versions (e.g. "`FORMAT` 4 requires
SST `format_version >= 5`"), or whether the three counters remain independently
incremented indefinitely. Separately unresolved: what changed at each prior `FORMAT`
version, and whether any of that history concerned the manifest's own encoding rather
than the SST or WAL encodings.

## 3. Terminology

| Term | Meaning |
|---|---|
| Manifest | The full logical state: file set, column-family set, and durability counters, as reconstructed by loading a snapshot (or legacy mirror, or default) and replaying journal edits on top. |
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

Unlike the WAL frame header, the record header here has no fixed cap on
`payload_len` documented at the framing layer (**TODO: verify** whether an implicit
cap exists via the underlying JSON payload or file-size limits).

### 4.2 Record type registry

| Value | Meaning |
|---|---|
| 1 | `AddSst` edit |
| 2 | `RemoveSst` edit |
| 3 | `CreateColumnFamily` edit |
| 4 | `DropColumnFamily` edit (legacy form, without a captured file set — see §5.3) |
| 5 | `BumpWalSeq` edit |
| 6 | `BumpNextSstSeq` edit |
| 7 | `SetCloudCheckpoint` edit |
| 8 | `Batch` (a group of edits applied atomically as one journal record) |
| 9 | Fsync marker (not an edit; see §4.4) |
| 10 | `DropColumnFamilyAt` edit (drop with captured snapshot frontier + file set) |
| 11 | `ReclaimColumnFamily` edit |

Any `record_type` outside `1..=11` is corruption (§6.1). This is a closed, currently
fully-enumerated set — there is no reserved gap or "unknown, skip" tolerance the way
the WAL's TLV tags are individually skippable (`wal.md` §4.1); an unrecognized
manifest journal record type is always a hard error, never silently ignored. This
makes the manifest journal's forward-compatibility story stricter than the WAL's: a
new edit kind requires every reader to be upgraded before it can appear in a shared
journal, whereas the WAL's TLV scheme lets old readers skip new optional tags.

### 4.3 Payload encoding: an edit envelope, not a fixed binary layout

Unlike the WAL's payload (`wal.md` §4, a fixed magic+version+TLV binary scheme), a
manifest journal payload is a **JSON-serialized value** — specifically, an "edit
envelope":

```jsonc
{
  "edit_id": <u64>,
  "edit": { /* tagged union matching one ManifestEdit variant, by name */ }
}
```

- `edit_id`: the durable, monotonically increasing identity for this record (§3).
- `edit`: the `ManifestEdit` payload, serialized as an **externally tagged** JSON
  union — `{"VariantName": <fields>}`. For example `{"RemoveSst": {"name": "..."}}`,
  `{"AddSst": {<FileMeta fields, inlined>}}` (a newtype variant inlines its single
  field's own object), and `{"Batch": [<edit>, ...]}`. An implementation must
  reproduce this shape exactly, not one of the other conventional enum-to-JSON
  encodings (internally tagged, adjacently tagged, or untagged).

A reader must additionally accept a **legacy payload shape**: the bare `ManifestEdit`
JSON value with no `edit_id`/`edit` wrapper at all. When a payload fails to parse as
the envelope shape, a reader falls back to parsing it as a bare edit and synthesizes
an edit id one greater than the highest seen so far in the replay, computed
incrementally during that same forward scan rather than by a prescan of the file. The
synthesized id is therefore deterministic for a fixed journal, but derives from scan
position rather than being a stable persisted identity. This keeps journals written
before the envelope wrapper existed readable. A conforming writer always appends the
enveloped shape; the bare shape is a read-only compatibility target.

`ManifestEdit` variants and their fields (field names as JSON keys; all integers
JSON numbers, not strings):

| Variant | Fields | Semantics |
|---|---|---|
| `AddSst` | one `FileMeta` value (§5.5) | Add (or replace, by name) one SST file's metadata. |
| `RemoveSst` | `name: string` | Remove the SST with this exact name, if present. |
| `CreateColumnFamily` | `id: u32`, `name: string`, `created_at: u64` (millis since Unix epoch) | Create (or, if `id` matches a previously-deleted tombstone, resurrect) a column family. |
| `DropColumnFamily` | `id: u32` | Soft-delete a column family; legacy form with no captured drop frontier or file list (see §5.3). |
| `DropColumnFamilyAt` | `id: u32`, `drop_sequence: u64`, `dropped_sst_names: [string]` | Soft-delete a column family, recording the sequence frontier and the exact SST names owned by it at drop time, so reclamation is retryable. |
| `ReclaimColumnFamily` | `id: u32`, `names: [string]` | Physically remove the named SSTs (previously captured by a drop) and mark the family's tombstone reclaimed once its captured file list is empty. |
| `BumpWalSeq` | `seq: u64` | Advance `last_persisted_sequence` if `seq` is greater than the current value (monotonic max, not overwrite). |
| `BumpNextSstSeq` | `cf_id: u32`, `next_seq: u64` | Advance the per-CF next-SST-sequence counter, monotonic max. |
| `SetCloudCheckpoint` | one `CloudCheckpoint` value: `checkpoint_sequence: u64`, `covering_ssts: [string]` | Advance the cloud durability checkpoint, monotonic max on `checkpoint_sequence`. |
| `Batch` | `[ManifestEdit]` (a list, possibly recursively containing another `Batch` — see below) | Apply every listed edit, in order, as a single atomically-recorded journal entry. |

All SST names appearing in any edit (`AddSst.name`, `RemoveSst.name`,
`dropped_sst_names`, `ReclaimColumnFamily.names`, `covering_ssts`) must independently
validate as a **persisted SST name** before the edit is accepted for either append or
replay: a single path component, relative, containing no `/`, `\`, `:`, or NUL bytes,
ending in a `.sst` extension. A name failing this check is corruption — this guards
manifest-driven filesystem joins against path traversal regardless of where the name
came from (a hand-edited journal, a corrupted record that still parses as valid
JSON, etc).

Applying `apply_edit` for each variant is defined to be **idempotent-safe under
replay** in the sense that re-applying an already-applied edit produces the same
state (e.g. `AddSst` replaces-by-name rather than appending a duplicate;
`BumpWalSeq`/`BumpNextSstSeq`/`SetCloudCheckpoint` take a monotonic max rather than
an unconditional set). This matters because a crash between "snapshot renamed
durable" and "journal truncated" can otherwise cause the same edit to be visible via
both the new snapshot and a stale journal tail (see §6.3), and idempotent apply
semantics on every edit variant are what make that race harmless rather than a
consistency bug.

**Note on `SetCloudCheckpoint`'s comparison operator:** `SetCloudCheckpoint` advances
on `>=`, not the strict `>` used by `BumpWalSeq` and `BumpNextSstSeq`. A replayed edit
whose `checkpoint_sequence` *equals* the current value therefore still overwrites
`covering_ssts`. The sequence itself remains monotonically non-decreasing, but
replaying an equal `checkpoint_sequence` carrying a different `covering_ssts` list is
not a true no-op — the later-applied edit wins. **TODO: verify** whether this
asymmetry is deliberate (allowing `covering_ssts` for a given sequence to be refined
across edits) or should be tightened to match the other two counters.

### 4.4 Fsync markers and the durability boundary

A special record type (9) — not a `ManifestEdit` at all — is appended immediately
after every edit (or edit batch) record, as part of the same physical write/fsync
call:

```jsonc
{ "last_persisted_sequence": <u64>, "ts_millis": <u64> }
```

`last_persisted_sequence` here holds the **edit id** of the edit record the marker
follows (a naming carryover, not a WAL sequence number), and `ts_millis` is a
wall-clock timestamp for observability.

This marker exists because replaying JSON records with independent CRCs does not by
itself guarantee that an edit and its intended "this is durable" boundary landed
together on disk: a writer can crash after successfully appending the edit record's
bytes but before the following fsync, or after the fsync call was issued but before
it completed. The marker's presence, verified with its own CRC, is what lets a reader
distinguish "this edit was appended and then explicitly synced" from "this edit's
bytes happen to be present in the file but no sync confirmation followed it" (see
§6.2's stricter tail rule: an edit at EOF with no following marker is *not* treated as
a benign incomplete tail the way a WAL's last frame can be — it is dropped from the
replayed edit set, but a partial *marker* — bytes present but truncated/corrupt — is
what triggers the journal-repair path of §6.2).

Every `append_edit`/`append_edit_batch` call writes exactly one edit/batch record
followed by exactly one marker record, both flushed to durable storage (`fsync`) as
part of the same call before the call returns successfully — so, absent a crash, the
count of durable markers always equals the count of durable edit records.

## 5. Recovery: loading a manifest

### 5.1 Base-state precedence

On load, exactly one of the following supplies the starting (pre-journal-replay)
manifest state, in this precedence order:

1. **`manifest.snapshot.json`**, if it exists — parsed as the full JSON `Manifest`
   structure (§5.5/§5.6 below). This is authoritative whenever present; step 2 is not
   consulted at all if this file exists, even if it disagrees with `manifest.json`.
2. **`manifest.json`** (the legacy mirror), if no snapshot exists but this does.
3. An empty **default** `Manifest` (`last_persisted_sequence = 0`, no files, no
   column families, `next_wal_seq = 1`, empty counters, `edit_checkpoint_id = 0`), if
   neither file exists.

Both `manifest.json.tmp` and `manifest.snapshot.json.tmp` staging files are never
treated as authoritative sources at load time — only the final (non-`.tmp`) names are
read, so a crash mid-write of either leaves the previous durable state intact and the
stray `.tmp` file inert until the next write attempt overwrites or a cleanup routine
removes it.

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

After base-state loading and journal replay, every persisted SST name reachable from
the resulting manifest (each `FileMeta.name`, and every name inside
`cloud_checkpoint.covering_ssts`) is validated as a well-formed persisted SST name
(§4.3); a validation failure at this point fails the load, even if the underlying
JSON/journal parsing succeeded.

### 5.3 Column-family lifecycle

A column family's manifest record (`ColumnFamilyMeta`) moves through these states:

1. **Active**: `deleted_at` absent (null). Newly created via `CreateColumnFamily`.
   Ids are allocated as `1 + max(existing ids)`, saturating rather than wrapping at
   `u32::MAX`; id-space exhaustion is a hard error, not silent reuse.

   Id `0` conventionally denotes the always-present default column family, but that
   reservation is **not** a manifest-format rule. The next-id computation folds in
   no implicit `0`, id 0 never receives an explicit `ColumnFamilyMeta` record, and
   nothing in the persisted state prevents a `CreateColumnFamily{id: 0, ...}` edit
   from being replayed. An implementation wanting that guarantee must enforce it one
   layer up — rejecting attempts to create or drop id 0, or the name `"default"`, at
   its DDL layer — because the manifest format will not enforce it.
2. **Soft-deleted (tombstoned)**: `deleted_at` set to a millisecond timestamp via
   `DropColumnFamily`/`DropColumnFamilyAt`. The column family's id is **never
   reused** even after full reclamation (below) — the tombstone record persists
   indefinitely as the mechanism that prevents id reuse. `DropColumnFamilyAt`
   additionally records `drop_sequence` (the read-snapshot sequence frontier at or
   below which the family's data must remain visible to already-open snapshots) and
   the exact `dropped_sst_names` owned by the family at drop time, so which files
   must eventually be reclaimed is fixed at drop time rather than recomputed later
   (which could otherwise race with concurrent compaction publishing new files under
   the dropped CF's id).
3. **Reclaimed**: once every currently-open read snapshot's sequence number has
   advanced past `drop_sequence` (so no live reader can still observe the dropped
   family's data), a `ReclaimColumnFamily` edit removes the captured SST files from
   the manifest's file list (via the same `RemoveSst` path as an ordinary compaction
   removal) and clears the tombstone's `dropped_sst_names` to empty, at which point
   `reclaimed` is set `true`. The tombstone record itself (id, name, timestamps)
   remains in the manifest permanently — reclamation only removes the *file*
   entries, never the tombstone.
4. **Resurrection**: `CreateColumnFamily` for an id that currently has
   `deleted_at.is_some()` clears `deleted_at`, rewrites `name`/`created_at`, and
   reactivates the family — this is how a name can be recreated on an id that was
   previously dropped and reclaimed, without allocating a fresh id.

A family is only eligible to be listed as reclaimable once its `drop_sequence`
frontier has been passed by the oldest currently-pinned read snapshot (or
immediately, if there is no `drop_sequence` recorded at all — the legacy
`DropColumnFamily` path, which has no frontier concept and is treated as
"reclaimable whenever no snapshot floor is configured at all"). **TODO: verify**
whether the legacy `DropColumnFamily` (no captured frontier) path is still reachable
from any current writer, or purely a compatibility decode target for old journals —
new drops always appear to go through `DropColumnFamilyAt`.

### 5.4 File-set (version-edit) semantics

The manifest's file list (`files: [FileMeta]`) is a flat set, not partitioned by
column family or level at the storage level — `cf_id` and `level` are just fields on
each entry, and a reader reconstructs "files owned by CF X at level N" by filtering.
Reconstructing "the current file set" is pure last-writer-wins replay:

- `AddSst`: if an entry with the same `name` already exists, it is **replaced**
  in place (not duplicated); otherwise the new entry is appended. This makes
  `AddSst` safe to apply twice for the same file (e.g. if a batch containing it is
  ever replayed more than once under a bug, or by design for idempotent replay).
- `RemoveSst`: removes every entry matching `name` (in practice at most one, given
  `AddSst`'s replace-by-name behavior, but replay does not special-case "should be
  at most one").

There is no explicit "version" object distinct from the manifest's current file
list — unlike some other LSM manifest designs (e.g. LevelDB/RocksDB's `Version` +
`VersionEdit` model with an explicit level-sorted-run structure), this format's
"current version" is simply "the file list as of the most recently applied edit,"
recomputed by linear replay rather than through an explicit persisted version-number
object. **TODO: verify** whether any higher layer above the manifest module
maintains an equivalent to a RocksDB-style `Version` in memory — the manifest format
itself only commits to the flat replace/remove-by-name semantics above.

### 5.5 File metadata fields

Each `FileMeta` entry:

| Field | Type | Required in JSON? | Semantics |
|---|---|---|---|
| `name` | string | yes | Persisted SST filename; see §4.3's validation rule. Conventionally `{cf_id:06}_{level:02}_{sst_seq:020}.sst` (zero-padded decimal fields), though the manifest format itself does not require any particular name shape beyond the general persisted-name validation — the fixed-width convention is an SST-subsystem naming policy, not a manifest-format requirement. **TODO: verify** whether any manifest-level logic (e.g. sort order assumptions) actually depends on this naming convention, as opposed to only display/debugging convenience. |
| `level` | u32 | yes | LSM level the file belongs to. |
| `size_bytes` | u64 | yes | File size in bytes. |
| `content_crc32c` | u32, optional | no | Whole-file CRC32C, when recorded, used to validate the SST's bytes independent of the manifest journal's own CRC-32 (§4.1) — this checksum is over the *referenced SST file's content*, a different algorithm and a different object than the journal record checksum. |
| `cf_id` | u32 | no (defaults to 0) | Owning column family id. |
| `sst_seq` | u64 | no (defaults to 0) | Per-CF monotonic sequence number identifying this file's creation order within its column family; source of the `next_sst_seqs` counter advanced by `BumpNextSstSeq`. |
| `smallest_key` / `largest_key` | byte string, optional | no | Inclusive key bounds covered by the file, when known. Keys are arbitrary bytes, so their JSON representation must be pinned down: **TODO: verify** whether this is a JSON array of byte values or a base64 string. `schema/manifest.schema.json` accepts either pending that decision. Until it is settled, two implementations can fail to round-trip these fields against each other. |
| `smallest_seq` / `largest_seq` | u64, optional | no | Inclusive sequence-number bounds of entries in the file, when known. |
| `sublevel` | u32 | no (defaults to 0) | Sub-ordering within `level` (e.g. for L0 overlap ordering); semantics beyond "lower sorts first within the level" are **TODO: verify** — not established from the manifest module alone. |

A read-access counter (`read_count`) exists on the in-memory structure for
compaction-prioritization heuristics but is explicitly excluded from serialization
(`#[serde(skip)]`) — it is pure runtime state, never persisted, and out of scope for
this format document.

### 5.6 Manifest-level (top-level) fields

| Field | Type | Semantics |
|---|---|---|
| `last_persisted_sequence` | u64 | Highest data-plane (WAL) sequence number known to be durably reflected in the current file set; advanced only monotonically (max) via `BumpWalSeq`. |
| `files` | `[FileMeta]` | The flat file set; see §5.4. |
| `column_families` | `[ColumnFamilyMeta]` | All column families ever created, including tombstoned ones; see §5.3. |
| `cloud_checkpoint` | `CloudCheckpoint`, optional | Cross-references the WAL's cloud-durability layer (`wal.md` §1.2): `checkpoint_sequence` (highest WAL sequence fully materialized to cloud) and `covering_ssts` (the SST names that make that sequence durable without needing the WAL). Omitted entirely from JSON when absent, rather than serialized as null. |
| `next_wal_seq` | u64, default `1` | Next WAL sequence number to hand out; not the same field as `last_persisted_sequence` — this is a forward-looking allocator counter, not a durability watermark. |
| `next_sst_seqs` | map `u32 -> u64` | Per-column-family next-SST-sequence allocator counters, keyed by `cf_id` rendered as a canonical unpadded decimal string (JSON object keys are always strings); entries absent from the map default to `0` (i.e. next allocation would be sequence `0`, or whatever the SST layer's own base is — **TODO: verify** the exact base value a missing map entry implies for a fresh CF). |
| `edit_checkpoint_id` | u64, default `0` | The journal replay horizon this snapshot already reflects; see §5.2. Absent in manifests written before this field existed, which is why it defaults to `0` (replay everything) on decode — a conservative default that trades a possibly-redundant-but-harmless full replay for never silently skipping edits an old snapshot doesn't actually contain. |

The whole `Manifest` structure is serialized as pretty-printed JSON (not a binary
format) for both the snapshot and the legacy mirror files — a deliberate
human-debuggability trade-off noted directly in the source, unlike the journal's
per-record binary framing (§4.1) which wraps a JSON payload but is not itself
freeform JSON.

## 6. Recovery semantics: corruption and torn tails

### 6.1 Hard corruption in the journal

Any of the following, encountered while scanning `manifest.journal`, is corruption
and is never silently tolerated as a benign tail (contrast §6.2):

- A `record_type` byte outside `1..=11`.
- A CRC-32 mismatch between a record's declared checksum and the checksum computed
  over its actual payload bytes.
- A record whose JSON payload fails to parse as either the edit-envelope shape or the
  legacy bare-edit shape (§4.3).
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
4. Rewrite `manifest.json` (the legacy mirror) to match.

A crash between steps 2 and 3 leaves a durable snapshot **and** an untruncated
journal whose already-merged edits are still physically present. This is exactly the
scenario §5.2's checkpoint-gated replay is designed to make harmless: the new
snapshot's `edit_checkpoint_id` already reflects those edits, so replay skips them
even though their bytes remain on disk; a subsequent successful checkpoint operation
truncates the journal for real. A crash before step 2's rename completes leaves the
old snapshot (or no snapshot) and the full journal intact — ordinary §5.1/§5.2
recovery applies unmodified. A concurrent competing checkpoint from a stale
in-memory manifest clone is resolved the same way: whichever checkpoint reaches
durable storage last simply carries forward the higher of the two `edit_checkpoint_id`
values and the durable-max of every monotonic counter, rather than overwriting
recorded durable state with a stale clone's lower values (see §7's "checkpoint base
merge" step).

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
  several edits in one journal record): both are format-legal (record types 1-7 vs.
  8), and a reader must handle either. Whether a given writer chooses to batch (e.g.
  to make a flush's "add these N new SSTs" atomic as one journal record) is a
  performance/atomicity-of-intent choice by the writer, not a format requirement.
  (Note this *is* format-relevant in one sense: choosing to batch several edits
  changes what atomicity a reader observes across a crash — but the *decision* of
  when to batch is policy; the *batch record's* structure and replay-atomicity
  guarantee is format, §4.3.)
- **Fsync/durability policy for journal appends**: this format requires an fsync of
  both the edit record and its
  marker before an append call returns — unlike the WAL, which exposes multiple
  durability policies (`wal.md` §7) that trade off when bytes are synced. **TODO:
  verify** whether synchronous durability is required by design here (the manifest
  being low-volume relative to the data WAL), or whether batched or asynchronous
  manifest durability is a legitimate future policy variation. Either way it would
  not change the on-disk format.
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
- **`RecoveryPolicy` (Strict vs. a more permissive "warn and continue with base
  state alone") for journal-replay failures (§5.2)**: like the WAL's
  Strict/SalvageValidPrefix split (`wal.md` §7), this is reader-side policy about
  the *response* to detected corruption, not about what counts as corruption. In
  scope: the corruption/tolerated-tail classification of §6.1–§6.2. Out of scope:
  which response a given deployment configures.

**Settled:**

- `ManifestEdit`'s tagged-union JSON shape is externally tagged
  (`{"VariantName": <fields>}`); see §4.3, which states the exact shape an
  implementation must reproduce.
- The frontier-less `DropColumnFamily` variant is a decode-only compatibility
  target: a conforming writer appends only `DropColumnFamilyAt`. This is unlike the
  WAL's legacy split-marker transaction encoding (`wal.md` §5.3a), which remains an
  actively written path.

**Open questions:**

- **TODO: verify** — Whether `manifest.json` (the legacy full-state mirror) is a
  permanent part of the format contract that every implementation must keep writing,
  or a deprecated compatibility artifact that a new implementation could in
  principle skip writing (while still reading it as a base-state fallback per
  §5.1) without breaking any consumer.
- **TODO: verify** — What an absent entry in `next_sst_seqs` for a given `cf_id`
  (§5.6) implies: which sequence number is allocated first for a column family that
  has never had a `BumpNextSstSeq` edit recorded. Until this is settled, two
  implementations can disagree on the first SST sequence for a fresh column family.
