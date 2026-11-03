# Cloud Metadata Authority Format

Status: draft. Requirements use *must*, *must not*, *required*, *should*, and
*may* in the sense of RFC 2119.

## 1. Profile and authority

Cloud metadata layout v2 uses lease document v4, initialization sentinel v3,
a separate binary committed pointer, and immutable manifest-version objects.
The WAL publication catalog remains v3 (`wal.md` §1.2). Legacy lease-embedded
metadata pointers and mutable metadata copies are incompatible recovery
authorities. A validated v4 pointer may still explicitly select immutable
per-file objects (§4.1); their mere presence is never authority. Logical export/import into a fresh prefix is required; there is
no in-place migration.

An upload, an object ordinal, LIST ordering, a local cache, or an object under
legacy `metadata/generations/` must not establish commitment. Only the
validated committed pointer selects recovery authority.

## 2. Lease and initialization sentinel

`midge_primary_lease.json` contains UTF-8 line-based text, in this writer order:

```text
lease_version: 4
fencing_epoch: <u64>
holder: <holder id>
owner: <unique owner token>
acquired: <RFC-3339 timestamp>
expires: <RFC-3339 timestamp>
```

Readers require a final newline, reject CR, duplicate/unknown fields and
missing fields, and require nonempty holder, owner and acquisition timestamp.
The epoch must be positive and increase on takeover. Unlike v3, the lease
contains no `metadata` line. Heartbeat renewal changes the lease's conditional
identity independently of the committed pointer.

`metadata/authority-initialized.v1` contains exactly one of:

```text
midge cloud metadata authority v3 pending
```

```text
midge cloud metadata authority v3 initialized
```

Both bodies end with one newline. Fresh initialization remains pending until
lease acquisition and pointer claim complete; only then may validity activate
and the sentinel become initialized. A later epoch may finish interrupted
pending initialization. An initialized sentinel with missing lease or pointer
must fail closed, never bootstrap empty authority. Legacy sentinel/lease
versions must not use the pending-initialization repair path.

## 3. Committed pointer and generation descriptor

`metadata/committed-pointer.v1` is one kind-4 frame (`metadata.md` §2). Its map
has `writer_epoch: u64`, `holder_id: string`, `owner_token: string`, and
`committed: Generation | nil`.

A generation map has `id: u64`, `manifest_horizon: u64`, and `objects: [Object]`.
The positive `id` is a shared publication ordinal, independent of the manifest
journal horizon. A successor is exactly the previous ordinal plus one, starting
at 1; its horizon must not regress. Republishing the identical descriptor is
idempotent. An object map has exactly `file_name: string`, `object_key: string`,
`len: u64`, and `crc32c: u32`. Length and CRC32C pin the entire encoded object,
including its frame checksum. Pointer epoch must be positive and holder/owner
nonempty. Generation and object maps reject unknown fields.

Before serving recovery or writes, a successor lease holder must CAS-claim
the pointer with its epoch, holder and owner token while preserving committed
predecessor history. A greater epoch, or equal epoch with a different owner,
must not be claimed. Claims and generation publication compare-exchange the
same pointer identity. A predecessor mutation either commits before the claim
and is retained by it, or loses its original condition after the claim. Reading
a successor identity must not authorize a stale writer to publish: the epoch
and owner must also match.

Upload receipts must validate the exact bytes before pointer CAS; absent or
ambiguous receipts require exact identity-bearing readback. A bare ETag is not
payload-integrity proof. Lost claim/publication responses may be reconciled
only by exact pointer readback under the same epoch and owner; timeout alone
does not cancel an admitted provider effect.

## 4. Immutable manifest versions

For a version-chain generation, `objects` lists its exact closure newest first,
with 1–16 entries and at most 64 MiB total encoded bytes. Every entry has
`file_name = "manifest-chain"` and key
`manifest/{ordinal:020}/{operation_uuid}`: a 20-digit zero-padded ordinal and
canonical lowercase hyphenated UUID. A version is one kind-5 frame with fields:

| Field | Type | Meaning |
|---|---|---|
| `id` | u64 | Ordinal matching its key and closure position. |
| `writer_epoch` | u64 | Positive producing epoch. |
| `operation_id` | string | UUID matching the key suffix. |
| `parent` | Object or nil | Exact next entry in the pointer's closure. |
| `changes` | map string to FileEdit | At most four unique known control filenames. |
| `cut` | optional CutProof | Exact capture/coalescence proof (§5). Omitted when absent. |

Known files are `FORMAT`, `manifest.snapshot.json`, `manifest.journal`, and
`intent_log.json`. `FileEdit` is externally tagged `Replace` with binary bytes,
`Suffix` with `{keep: unsigned integer, bytes: binary}`, or unit string `Remove`.
The oldest version has no parent and may contain only replacements: its map
defines the complete file presence set. Apply later versions oldest to newest.
`Suffix` retains `keep` bytes from an existing file and appends `bytes`; `keep`
must fit that file. `Remove` makes the file absent. Deltas without parents,
unknown or duplicate names, and invalid suffix prefixes are corruption.

Recovery must verify each object's exact length/CRC32C, kind, ordinal, UUID,
and parent reference before trusting the closure. Ordinals decrease by exactly
one down the closure and epochs must not increase toward older versions.
Reconstructed manifest, intent and FORMAT bodies must pass their own decoders
and semantic validation. The pointer's horizon must describe the reconstructed
durable manifest state; a high ordinal alone is not evidence of a high horizon.

Writers must upload immutable versions conditionally without overwriting
existing objects, then CAS the pointer. UUIDs separate uncommitted attempts
at the same ordinal. A checkpoint starts a new parentless complete snapshot;
it may discard history only from the new closure, never from readers or
in-flight publications that still pin the old one. The writer may checkpoint
at 16 objects, 16 KiB of edits, or an oversized individual edit; closure limits
are required independently of the chosen trigger.

### 4.1 Explicit immutable per-file generations

A pointer may instead select 1–4 objects with unique known control filenames,
including required `FORMAT` and `manifest.snapshot.json`. Each key must be
`metadata/generations/{canonical_uuid}/{file_name}` and its filename must match
the descriptor. The descriptor defines the complete presence set; an omitted
journal or intent file is absent. Validate exact lengths, CRC32C and decoded
contents before recovery. These references must not mix with `manifest-chain`
entries. This explicit pointer-selected representation does not authorize
choosing a legacy upload by LIST or reverting to a lease-embedded pointer.

## 5. Exact metadata-cut proof

A cut map has `writer_epoch: u64`, `session: string` (UUID), `ordinal: u64`,
`digest: [u8; 32]`, and `covers: [{ordinal: u64, digest: [u8; 32]}]`.
Epoch and ordinal must be positive. The epoch must equal the head version's
epoch. There are at most 63 witnesses, strictly ordered by positive ordinal,
each less than the cut ordinal. Digests are arrays of 32 unsigned bytes.

Compute SHA-256 by visiting the four control filenames in the order listed in
§4. For each, hash its UTF-8 name length as u64 big-endian, then its name bytes,
then one presence byte. For absence append `0`; for presence append `1`, the
body length as u64 big-endian and the exact body bytes. The cut digest must
match the reconstructed file presence set and bytes.

A proof covers another capture only in the same writer epoch and session,
by exact equality or an explicit earlier `(ordinal, digest)` witness. A higher
ordinal or manifest horizon alone must not prove an earlier capture committed.

## 6. DDL authority and retention

`metadata/ddl.registry.json` remains a separate JSON conditional authority.
Its v2 map has `registry_version: 2`, `ddl_epoch: u64`, positive
`writer_epoch: u64`, `column_families: [ColumnFamilyMeta]` and
`operations: [{op_id: string, edit: ManifestEdit}]`. Unknown top-level fields
are rejected. It deliberately omits the legacy required `epoch` field.
A successor must conditionally fence this registry before serving recovery;
a predecessor decision that commits first must be incorporated.

Cleanup must preserve the entire selected closure and every dependency held by
snapshots, readers, intents, compactions and in-flight publication. Age, LIST
results and an unreferenced upload alone do not prove safe deletion. Conditional
object identity and validated current authority are required. Failures and
ambiguous callbacks retain obligations for retry, including across restart.

An admitted DDL decision may finish after its caller's deadline. Lost responses
must report an unknown outcome and reconcile authority before retrying. After
an uncertain local DDL journal append, further writes/metadata changes must be
fenced until recovery resolves it; later acknowledged work must not depend on
guessing whether that append committed.

## 7. Format vs. policy

Authority selection, fencing, exact closure validation and dependency retention
are requirements. Heartbeat scheduling, checkpoint thresholds within the bounds,
retention grace periods, caches and internal task scheduling are policy.

## Open questions

None for the rules specified here. Cloud wire fixtures and schemas remain
incomplete; documentation is not a claim of cross-engine qualification.
