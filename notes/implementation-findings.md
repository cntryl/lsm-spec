# Implementation findings (non-normative)

This file is **not part of the specification.** Nothing here is normative, and no
document under `format/` depends on it.

The initial specification under `format/` was reconstructed by comparing two
independently built storage engines. Their code supplied historical and
interoperability evidence, not normative authority. The resulting specification is
the source of truth and may deliberately require behavior that one or both engines
do not currently implement. This drafting history is recorded here so `format/` can
state what the formats *are* without reference to any particular implementation.

The engines are referred to below as **engine R** (a Rust implementation) and
**engine C** (a C# implementation). Findings were current as of the audit that
produced spec version `0.1.0-dev`; they are not maintained in step with either engine
and should be re-verified before being relied on.

## 1. Divergences from the specification

Each entry is a place where an engine does not implement what `format/` requires. In
every case the specification was upheld and the engine treated as having a gap to
close — the specification is not rewritten to match an implementation.

### 1.1 Acquisition does not accept an epoch floor — engine R

`format/lease.md` §4 step 4 requires lease acquisition to accept a caller-supplied
minimum epoch and use it as a floor. This is what guarantees a newly granted epoch
exceeds every epoch already durable in the engine's own WAL, which is the property
WAL fencing (`format/wal.md` §6.4) depends on.

- **Engine C** implements it: its acquisition entry point takes a `minimumEpoch`
  parameter, computes `max(current epoch, minimumEpoch) + 1`, and threads the
  parameter out to a public argument on its top-level open API.
- **Engine R** does not. Its acquisition API has no equivalent parameter and no
  plumbing for one. Acquisition also completes in full *before* WAL replay runs, so
  the `max_epoch_seen` its replay computes is not reachable at the point it would be
  needed.

**Recommended fix (engine R):** add an equivalent minimum-epoch parameter to the
lease-acquisition and engine-open API surface, matching engine C's design, so that a
caller with out-of-band knowledge of a higher epoch — disaster recovery, a
multi-replica coordinator, a migration tool — can supply it.

**Caveat:** engine C's default for this parameter is `0`, and it also acquires the
lease before loading the manifest. Out of the box, with no caller supplying a real
value, engine C is exposed to the same hazard. Neither engine closes the gap
automatically; engine R merely lacks the option entirely.

Epoch exhaustion is handled correctly in both: each fails closed at the maximum
value rather than wrapping.

### 1.2 A corrupt leader record is treated as an absent one — engine R

`format/lease.md` §7 requires an empty, non-UTF-8, or field-incomplete leader record
to be treated as **indeterminate** (fail closed).

Engine R returns the same result for an empty or field-malformed record as for an
*absent* file — "no lease," an effective epoch baseline of 0 for the next acquirer —
so a corrupted record silently resets fencing state instead of blocking acquisition.
A non-UTF-8 record does fail closed, but as a generic I/O error rather than the
specific indeterminate classification.

**Recommended fix (engine R):** fail closed as indeterminate in all three cases.
Engine C's record reader already does this and can serve as the reference: it raises
its indeterminate error both when no line parses into any field and when any required
field is missing after parsing, never conflating either case with an absent file.

### 1.3 Hot-path fencing check omits `holder_id` — engine R

`format/lease.md` §6 item 1 requires a fencing check to compare both `epoch` and
`holder_id`. Engine R's hot-path check at WAL-sync boundaries compares only `epoch`.
Its acquisition-time and renewal-time checks compare both correctly; it is
specifically the sync-boundary check that is narrower than the contract.

Under the epoch-CAS protocol of §4, two holders should never legitimately share one
epoch, so this is not known to be exploitable through conforming writers alone. The
`holder_id` comparison exists as defense against what §4 does not cover: a tampered
or externally rewritten record, a non-conforming writer sharing the directory, or a
defect in another implementation's CAS logic.

**Recommended fix (engine R):** extend the sync-boundary check to compare
`holder_id` as well. Engine C's equivalent hot-path check already compares both.

### 1.4 Duplicate-field handling differs between engines

`format/lease.md` §7 specifies last-occurrence-wins for a duplicate field in the
leader record. Engine R implements this; engine C rejects duplicates as
indeterminate.

The specification adopts last-occurrence-wins, so engine C diverges as written. The
opposite choice is defensible and is recorded as an open question in
`format/lease.md` §8: no conforming writer can produce a duplicate field, since every
write replaces the whole file atomically, so a duplicate is itself evidence of
tampering — which is the situation the surrounding rules handle by failing closed.

### 1.5 `SetCloudCheckpoint` comparison operator — engine R

`format/manifest.md` §4.3 notes that engine R advances `SetCloudCheckpoint` on `>=`
while `BumpWalSeq` and `BumpNextSstSeq` use strict `>`. The sequence itself still
never regresses, but replaying an equal `checkpoint_sequence` with a different
`covering_ssts` is not a true no-op. Whether the asymmetry is deliberate is
unresolved; it is carried as an open question in the specification rather than as a
required fix.

## 2. Corrections where the specification was wrong

Two rules were withdrawn because the specification itself was incorrect, not because
an implementation failed to meet it.

### 2.1 Epoch-mixing within a local WAL file

An earlier draft stated that a WAL file containing records from more than one
`writer_epoch` is corrupt. This contradicted the append-on-reopen recovery model the
same document describes, and enforcing it would have broken ordinary post-failover
recovery.

Both engines reopen the active WAL file in place across a failover rather than
rotating it, and neither rejects epoch-mixing during local recovery. Engine R's
single-epoch validation is confined to helpers reachable only from the cloud
upload/download path.

The rule is now inverted (`format/wal.md` §6.4 point 3: epoch-mixing is normal and
must not be rejected), with the genuinely correct, narrower constraint stated
separately in §1.2 — an object *uploaded* under an epoch-scoped key must itself be
single-epoch.

### 2.2 Extended-length promotion boundary in SST entries

An earlier draft stated that `key_delta_len == 0xFFFF` is never a legitimate literal
length and that a writer must always promote to the extended form at exactly 65,535
bytes. An audit pass flagged engine R's writer as violating this.

The rule was too strict. The governing contract is the reader's **conjunctive**
check — extended form is in use only when `key_delta_len` and `value_len` both equal
their sentinels — under which a literal `0xFFFF` key-delta length with an ordinary
value length is valid and round-trippable. Both the rule and the finding against
engine R were withdrawn; `format/sst.md` §3.2 now states the minimal writer
requirement that follows from the reader's check.

## 3. Facts established by cross-implementation agreement

These were confirmed in both engines' source and are stated as plain requirements in
`format/` without attribution.

- **Lease file names** `.midge_leader` and `.midge_leader.lock`, constructed with
  these literal names in both engines.
- **Leader record line format** — both write `epoch`, `holder_id`, `acquired_at` in
  that order with a `": "` separator, though both parse field order tolerantly.
- **Mutation lock line format** — both write `field=value` with a bare `=`, distinct
  from the leader record's separator.
- **Release sentinel** — both write the identical literal `1970-01-01T00:00:00Z`
  rather than merely some sufficiently old timestamp, and neither deletes the record.
- **Version constants** — the directory-level format version (3), the SST footer's
  format version (4), and the WAL payload version byte (1) are three independent
  constants in engine R, with no code path deriving or cross-checking one from
  another.
- **Compression threshold** — both skip compression below a 256-byte input, a writer
  heuristic with no reader-visible effect.
- **`writer_epoch == 0`** is special-cased in engine R as a fencing-disabled
  sentinel, in both places that matter (recording a frontier and testing staleness),
  rather than being an unset-field default.
- **Split-marker transactions are still written**, not merely read: engine R falls
  back to them for transactions too large to buffer as one atomic frame.
- **Legacy forms never written by a current writer** — the `wal_{segment:06}.log`
  segment name and the frontier-less `DropColumnFamily` edit are decode-side only in
  engine R. Whether either was ever emitted by a superseded version is not
  determinable from current source.
- **Vestigial constants** — engine R declares a restart interval of 16 that is
  referenced nowhere outside its own declaration and governs no wire structure. The
  specification therefore describes no restart array at all.
- **`Merge` entries and value-bearing `Delete` entries** are structurally decodable
  but unreachable from any writer path in engine R, which is why `format/sst.md`
  reserves both.
- **Column-family id 0** is not reserved at the manifest level in engine R; the
  reservation is enforced one layer up, in its DDL layer.
- **Footer decode order** — engine R attempts a full footer decode first whenever the
  file is long enough, falling back to the bare-magic legacy check only on failure.

## 4. Questions the source could not answer

Carried into `format/` as open questions:

- What changed at each earlier directory-level format version, and whether any of it
  concerned the manifest encoding specifically.
- Whether a merge-operator concept exists in engine C. If so, the `Merge` reservation
  should be lifted in favor of a real specification.
- Whether engine C produces a value-bearing `Delete` entry via a path not examined.
- Whether byte-for-byte SST footer compatibility with the well-known engine the magic
  constant is borrowed from is an actual goal or incidental reuse.
- Whether the per-block Bloom filter's inner serialization is meant to be locked as a
  cross-engine format or left a swappable accelerator detail.
- How `smallest_key`/`largest_key` are represented in manifest JSON (byte array
  versus base64 string).

## 5. Maturity note

Engine C is the earlier-stage of the two. Rough edges observed in it — not
distinguishing a compatibility error from corruption on SST footer failure, not
validating every manifest-carried SST name at replay time, and having no
manifest-journal repair-on-next-write mechanism — are recorded as the current state
of an in-progress implementation, not as confirmed specification violations, and no
fixes are prescribed for them here.
