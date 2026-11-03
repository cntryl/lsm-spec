# Binary Metadata and Intent Format

Status: draft. Requirements use *must*, *must not*, *required*, *should*, and
*may* in the sense of RFC 2119.

## 1. Scope and compatibility

FORMAT 9 uses a shared binary codec for local and cloud metadata. Historical
filenames ending in `.json` do not imply JSON contents. Readers must not fall
back to JSON when binary decoding fails. FORMAT 8 data requires logical export
and import into a new database; no in-place upgrade or binary rollback exists.

The marker, WAL, SST and metadata version gates are independent. WAL payload,
ledger and segment versions remain 2; cloud WAL catalog version remains 3.
SSTs require V5. These unchanged WAL versions do not imply compatibility with
an older database marker or cloud metadata authority layout.

## 2. Shared frame

| Offset | Bytes | Field |
|---|---|---|
| 0 | 8 | ASCII magic `MDGEMD09`. |
| 8 | 1 | Kind from the registry below. |
| 9 | 4 | MessagePack payload length, unsigned little-endian. |
| 13 | length | Exactly one MessagePack value. |
| 13 + length | 4 | CRC32C, big-endian, over every preceding frame byte. |

Total frame size is `17 + length`. Payload length must not exceed 256 MiB.
The big-endian checksum is deliberate: the checksum over the complete encoded
object, including this field, must remain content-dependent for object identity
checks. Lengths stay little-endian; MessagePack's own multibyte primitives use
its standard big-endian encoding.

| Kind | Payload |
|---|---|
| 1 | Full manifest map (`manifest.md`). |
| 2 | Manifest journal edit envelope or fsync-marker map (`manifest.md` §4). |
| 3 | Intent suffix-edit map (§3). |
| 4 | Cloud committed-pointer map (`cloud-metadata.md` §3). |
| 5 | Immutable cloud manifest-version map (`cloud-metadata.md` §4). |

Readers must check the expected kind, complete frame extent and CRC32C before
applying payload semantics. A mismatching magic is `CompatibilityError`.
Wrong kind, trailing bytes, a checksum mismatch or an incomplete required
single frame is corruption. A payload length above the bound is corruption,
even if the announced bytes have not arrived. An incomplete magic/header is
a partial append only when its available magic bytes match the required magic
prefix and the containing log permits a torn final append.

Payloads use named maps with string field names, externally tagged enum maps,
compact integers, arrays for sequences/tuples and `nil` for absent options.
Unit enum variants use their variant-name string. Manifest SST key bounds are
MessagePack binary values, not hex strings or integer arrays. Per-family
allocator maps use integer keys. JSON Schema files describe decoded projections:
key bounds become lowercase hex and integer map keys become decimal strings.
MessagePack map iteration order is not a semantic identity; exact bytes remain
significant wherever a checksum, digest or object receipt pins them.

Before typed allocation, readers must validate that the payload contains one
complete MessagePack value, with no trailing values, reserved marker `0xc1`,
truncated strings/binaries/extensions, or container-count/extent overflow.
Root depth is zero; values at depth 64 or greater are rejected. Container child
counts must fit the remaining payload before allocating from their size hints.
Implementations may enforce additional resource budgets, but budget exhaustion
must not be treated as proof of corrupt or disposable authority.

## 3. Append-only intent suffix edits

`intent_log.json` is a concatenation of kind-3 frames. Each payload is a map:

| Field | Type | Meaning |
|---|---|---|
| `sequence` | u64 | Consecutive identity starting at 1. |
| `previous_checksum` | u32 | Prior frame's stored CRC32C; zero for the first edit. |
| `keep` | unsigned integer | Number of existing intent entries to retain. |
| `append` | array of intent entries | Replacement suffix after that retained prefix. |

Begin with an empty intent set, sequence zero and checksum zero. Each edit
must increment sequence by exactly one without overflow, match the preceding
checksum, and have `keep` no greater than the current set length. Truncate to
`keep`, append the suffix and adopt the new frame checksum. A missing or empty
file means an empty set. Writers retain the longest unchanged prefix when
recording a new set; clearing the set must not resurrect old intents.

Only a genuine incomplete final frame may retain the preceding valid state.
A complete invalid frame or a broken predecessor/prefix chain fails recovery
under both Strict and Salvage. Before the next append, durably replace a torn
file with its verified prefix using `intent_log.json.tmp` and atomic rename.
Append and durably sync the next edit; sync the directory when creating the
file. A checkpoint atomically replaces the log with a sequence-1 edit containing
the entire current set, `previous_checksum = 0` and `keep = 0`. Checkpoint
frequency is policy; a writer may use 1 MiB or 1,024 records as triggers.

### 3.1 Intent entry registry

Entries are externally tagged maps `{VariantName: fields}`:

| Variant | Fields |
|---|---|
| `SeqnoAllocated` | `seqno: u64`, `cf_id: u32` |
| `FlushPlanned` | `cf_id: u32`, `seqno_range: [u64, u64]` |
| `CompactionPlanned` | `input_files: [string]`, `output_level: u32` |
| `FlushPublish` | `phase`, `cf_id: u32`, `sequence: u64`, `file_meta: RuntimeFileMeta` |
| `CompactionPublish` | `phase`, `cf_id: u32`, `removed: [string]`, `added: [RuntimeFileMeta]` |
| `MoveSstsPublish` | `phase`, `captured: [FileMeta]`, `target_level: u32` |
| `SstAdded` | `file_meta: RuntimeFileMeta` |
| `CompactionApplied` | `removed: [string]`, `added: [RuntimeFileMeta]` |
| `WalSynced` | `segment_id: u64`, `seqno: u64` |
| `CloudUploadComplete` | `resource: string`, `seqno: u64` |

`phase` is the string `OutputDurable` or `ManifestPublished`. Intent phases
alone do not prove publication; recovery must consult authoritative metadata.
Every SST name must satisfy `manifest.md` §4.3, including captured, removed and
planned input names. A logical move's captured files are existing objects,
never disposable unpublished outputs.

`FileMeta` is the manifest type. `RuntimeFileMeta` has `name`, `level`,
`size_bytes`, optional `content_crc32c`, `cf_id`, nullable `smallest_key`,
`largest_key`, `smallest_seq`, `largest_seq`, default-false
`key_bounds_complete`, and optional `earliest_expiration_millis`; it has no
`sst_seq` or `sublevel`. Its key bounds use arrays of unsigned bytes, unlike
the manifest type's binary bounds. Absent optional checksum/expiration fields
are omitted. All other runtime fields are written, including null bounds.

## 4. Format vs. policy

Frame bytes, checksum order, kind separation, resource safety, intent-chain
validation and repair-before-append are format requirements. Checkpoint
frequency, buffer reuse, admission implementation and scheduling are policy.

## Open questions

None for the rules specified here. Binary fixtures do not yet cover every
intent variant or corruption case; see `conformance/README.md`.
