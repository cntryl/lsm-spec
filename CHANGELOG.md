# Changelog

## [Unreleased] — 0.1.0-dev

### Added
- `format/wal.md` — WAL framing, payload encoding, transaction semantics,
  recovery/torn-tail rules, writer-epoch fencing.
- `format/value-encoding.md` — shared compression-algorithm ID enum.
- `format/sst.md` — SST block/index/footer layout.
- `format/manifest.md` — manifest structure and version-edit semantics.
- `format/lease.md` — single-writer file lease format.
- `conformance/README.md` — conformance requirements.

### Changed
- `README.md` and `AGENTS.md` now make the authority direction explicit: existing
  implementations are research evidence used to reconstruct the formats, while this
  specification is the normative source of truth implementations must follow.
- Every `format/*.md` document now defines its rules in its own terms, without
  reference to any particular implementation. Earlier drafts cited implementation
  behavior directly, in place of or alongside the normative rule; that framing has
  been removed throughout. Where an earlier passage stated a rule *because* some
  implementation happened to do it that way, the rule is now stated as a
  requirement on its own terms, or — where no such rule could be justified on its
  own — marked `TODO: verify` instead.
- Document status headers now state RFC 2119 usage (`must`/`must not`/`required`/
  `should`/`may`) once, instead of repeating implementation-provenance language in
  every file.
- `conformance/README.md` now states plainly that no fixtures are published yet,
  and adds notes for fixture authors on schema limitations (see **Schema**, below).

### Fixed
- `format/wal.md` §5.4 — corrected the nested-transaction-batch minimum-record-length
  itemization, which omitted the mandatory `value`/`range_end` presence flags and so
  summed to 18 instead of the stated 20 bytes.
- `format/wal.md` §1.1 — reworded the active-file naming description so "always named
  `wal.log`" no longer directly contradicts the following paragraph's description of
  the legacy `wal_{segment:06}.log` naming pattern.
- `format/wal.md` §1.2.1 — specified that the publication catalog's `segments` map
  keys are canonical unpadded decimal (not the zero-padded form used elsewhere in the
  same section), and added the missing validity constraint that a map key must equal
  its entry's own `segment_id` field.
- `format/wal.md` §1.2, §1.2.1 — generalized the cloud object-key layout from a
  hardcoded `wal/` prefix to a deployment-configurable `{root}` (default `wal`), and
  noted that a validator must compare `object_key` against the root the catalog was
  actually loaded under.
- `format/sst.md` §7.1 — rewrote the footer-decode step as an explicit, ordered
  decision tree; previously "a magic mismatch...is `Corruption`" and the legacy
  V1–V3 `CompatibilityError` exception both claimed the same failure case with no
  stated precedence.
- `format/sst.md` §5.1 — corrected the block-handle validation floor from
  `size >= 4 + BLOCK_TRAILER_SIZE` read ambiguously to an explicit "at least 9
  bytes," and fixed a stray reference to a nonexistent §2.3.
- `format/sst.md` — realigned four ASCII-art block/frame diagrams whose column
  borders did not match their content-row widths (`wal.md` §3, §4.1, §5.4;
  `manifest.md` §4.1).
- `format/manifest.md` §2 — corrected the claim that the directory-level `FORMAT`
  integer, the SST footer's `format_version`, and the WAL payload's `version` byte
  are versioned together, since their current values (3, 4, 1 respectively) don't
  correspond and no arithmetic relationship between them is defined anywhere.
- `format/manifest.md` §5.6, `schema/manifest.schema.json` — clarified that
  `next_sst_seqs` map keys are `cf_id` rendered as a canonical unpadded decimal
  string (JSON object keys are always strings; the field description previously
  left this implicit).

### Corrected (the earlier text was wrong, not merely unconfirmed)
- `format/wal.md` §6.4 point 3 — this document previously stated that epoch-mixing
  within one physical WAL file is corruption. That directly contradicts this
  format's own append-on-reopen recovery model: the active file is never rotated at
  an epoch boundary, so an ordinary crash-then-failover cycle routinely produces one
  `wal.log` (and potentially a sealed segment) spanning more than one `writer_epoch`.
  The rule is reversed: epoch-mixing in a local file is normal and must not be
  rejected; §6.4 point 2's staleness rule is the sole mechanism for resolving it. The
  narrower, correct constraint — that an object *uploaded* to cloud storage under an
  epoch-scoped key must itself be single-epoch — now lives in §1.2, which this
  correction does not affect.
- `format/sst.md` §3.2/§3.3 — the extended-length-block promotion rule previously
  read "`key_delta_len == 0xFFFF` is never a legitimate literal length; a writer
  must always promote at exactly 65,535 bytes." That rule was stricter than the
  format's own reader contract requires: the governing check is conjunctive (extended
  form is in use only when `key_delta_len` **and** `value_len` both equal their
  sentinel values), so a literal `0xFFFF` key-delta length with an ordinary value
  length is valid and round-trippable. §3.2/§3.3 now state the minimal writer
  requirement that follows from the reader's conjunctive check.
- `format/lease.md` §3.3 — the mutation lock file's separator is `=`
  (`field=value`), not `": "` as previously stated; it does not share the leader
  record's line style. This is a description fix, not a design decision.

### Clarified (previously open, now settled)
- `format/manifest.md` §4.3 — `ManifestEdit`'s JSON shape is externally tagged
  (`{"VariantName": <fields>}`); the bare/legacy edit shape (no `edit_id`/`edit`
  wrapper) is a read-only decode target, never written by a conforming writer.
- `format/manifest.md` §5.3 — column-family id `0` denoting the default column
  family is not enforced by the manifest format; nothing in persisted state
  prevents a `CreateColumnFamily{id: 0, ...}` edit from being replayed. An
  implementation wanting that guarantee enforces it one layer up, at its DDL layer.
- `format/manifest.md` §5.3 — the frontier-less `DropColumnFamily` variant is a
  decode-only compatibility target; a conforming writer appends only
  `DropColumnFamilyAt`. Unlike the WAL's legacy split-marker transaction encoding
  (below), this path is not an actively written fallback.
- `format/wal.md` §6.4/§7 — `writer_epoch == 0` is a reserved sentinel meaning
  "fencing disabled," carried forward as a normative reserved value rather than an
  unset-field default.
- `format/wal.md` §5.3 — the legacy split-marker transaction encoding
  (`TxnBegin` … `TxnCommit`) is not a read-only compatibility path: it remains an
  actively written fallback for transactions too large to encode as a single atomic
  `TxnBatch` frame (the "spill" case). A writer targeting full interoperability must
  be able to *emit* split-marker transactions, not merely decode them.
- `format/wal.md` §1.1 — the legacy `wal_{segment:06}.log` active-file naming
  pattern is recognized on decode only; no conforming writer produces it.
- `format/sst.md` §3.3/§9 — this format defines no restart points and no
  reader-visible restart-offset table. A writer's internal restart interval, if it
  keeps one, has zero effect on wire bytes; the `RESTART_INTERVAL` terminology entry
  has been removed as non-normative.
- `format/lease.md` §1 — an OS-level advisory lock is not required alongside the
  leader record; the leader record alone is the specified mechanism.
- `format/lease.md` §5.3 — release forces `acquired_at` to the exact sentinel
  `1970-01-01T00:00:00Z`, not merely some sufficiently old timestamp, and must not
  delete the record.
- `format/lease.md` §7 — a duplicate field in the leader record is not corruption;
  the last occurrence in the file wins. (Recorded as a considered trade-off in §7,
  with a design note on the alternative of failing closed instead.)
- `format/lease.md` §8, `format/wal.md` §7 — the lease's `epoch` and the WAL's
  `writer_epoch` are the same value by construction: a successful acquisition's
  `epoch` is copied once into `writer_epoch` at startup, before WAL replay runs, and
  is never reassigned. In cloud deployments the same value also seeds the WAL
  catalog's `fencing_epoch` (`wal.md` §1.2, §1.2.1).
- `format/value-encoding.md` §2 — the 256-byte minimum-compression-input-size
  convention is documented as a shared, non-normative writer heuristic common to the
  WAL value path and the SST block path; a reader must accept a compressed or
  uncompressed value of any size regardless.

### Requirements added (previously absent or only implicit)
- `format/lease.md` §4 step 4 — acquisition **MUST** accept a caller-supplied
  minimum epoch, and a recovering engine **MUST** supply one, computed from its own
  WAL/manifest recovery pass, before treating itself as open for new writes. Without
  this floor, an acquisition that only computes `current_lease_epoch + 1` can grant
  an epoch that is not actually higher than epochs already durable in this engine's
  own WAL whenever the leader record and the WAL have diverged (restore from an
  older backup, manual recovery, directory migration) — silently reintroducing the
  stale-writer ambiguity fencing exists to prevent.
- `format/lease.md` §5.1 — the fencing check (§6 item 1: compare both `epoch` and
  `holder_id`) applies on every write path a holder uses to confirm it still holds
  the lease, not only inside the renewal routine — explicitly including any hot path
  checked immediately before a durable WAL sync.
- `format/lease.md` §7 — an empty, non-UTF-8, or field-incomplete leader record must
  fail closed as **indeterminate**, and must not be treated the same as an *absent*
  file. An absent file legitimately resets the epoch baseline to 0; a
  present-but-corrupt record does not, since the true baseline is unknown.
- `format/wal.md` §5.1 — `Delete` records **MUST NOT** carry `VALUE` (or, therefore,
  `COMPRESSION`); a `Delete` record carrying either is corrupt. `sst.md` §3.4's
  reservation of a value-bearing SST `Delete` entry depends on this rule.
- `format/wal.md` §5.4 — the nested `TxnBatch` payload is never compressed:
  `COMPRESSION` applies only to a value-write record's `VALUE`, so a `TxnBatch`
  record carrying that tag is corrupt.

### Reserved (decided, not left open)
- `format/sst.md` §3.2/§9 — `Merge` (`entry_type = 3`) is reserved as of this spec
  version. No merge-operator contract exists anywhere in this specification, and no
  WAL operation produces one; a conforming writer must not emit it until a future
  revision defines merge-operator semantics.
- `format/sst.md` §3.4/§9 — a `Delete` entry with `value_len > 0` is reserved as of
  this spec version, pending a defined use case.

### Schema
- `schema/sst.schema.json` — corrected the footer `magic` constant's underscore
  grouping to match `sst.md` §7 exactly; rejects `Merge` entries and value-bearing
  `Delete` entries as reserved states that must not appear in conforming-writer
  fixtures, even though a reader must still decode them.
- `schema/manifest.schema.json` — `smallest_key`/`largest_key` now accept either a
  base64 string or a byte-value array, pending resolution of the open question in
  `manifest.md` §5.5 on which encoding a writer actually produces; `next_sst_seqs`
  keys are now constrained to the canonical unpadded-decimal pattern used
  elsewhere in this format.
- `schema/wal-catalog.schema.json` — `object_key`'s pattern now allows the
  deployment-configurable root prefix (`wal.md` §1.2) instead of hardcoding `wal/`.

### Conformance
- `conformance/README.md` — added notes for fixture authors: u64 upper-bound
  constraints are not enforceable by JSON Schema validators using IEEE 754 doubles
  and must be checked by the harness; several cross-field equality rules (WAL
  catalog key/entry consistency) are likewise out of schema scope; and the
  repository's own `.gitignore` excludes `*.log`, which would silently exclude a
  WAL fixture literally named `wal.log` unless the fixture directory is exempted.
