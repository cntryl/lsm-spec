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

### Fixed
- `format/wal.md` §5.4 — corrected the nested-transaction-batch minimum-record-length
  itemization, which omitted the mandatory `value`/`range_end` presence flags and so
  summed to 18 instead of the stated 20 bytes.
- `format/wal.md` §1.1 — reworded the active-file naming description so "always named
  `wal.log`" no longer directly contradicts the following paragraph's "legacy
  active-file naming convention" for `wal_{segment:06}.log`.
- `format/wal.md` §1.2.1 — specified that the publication catalog's `segments` map
  keys are canonical unpadded decimal (not the zero-padded form used elsewhere in the
  same section), and added the missing validity constraint that a map key must equal
  its entry's own `segment_id` field.
- `format/sst.md` §7.1 — rewrote the footer-decode step as an explicit, ordered
  decision tree; previously "a magic mismatch...is `Corruption`" and the legacy
  V1–V3 `CompatibilityError` exception both claimed the same failure case with no
  stated precedence.
- `format/manifest.md` §2 — corrected the claim that the directory-level `FORMAT`
  integer, the SST footer's `format_version`, and the WAL payload's `version` byte
  are "versioned together," since their current values (3, 4, 1 respectively) don't
  correspond and no arithmetic relationship between them is defined anywhere.

### Verified against the Rust reference implementation ("midge")
A conformance audit compared every claim above against `cntryl/midge`'s actual
source. **This spec is the source of truth for the format; midge is one
implementation of it, not the other way around.** Every finding below was
evaluated on that basis: where midge disagrees with a sound normative rule, the rule
stands and midge is flagged as having a gap to close — the spec is not rewritten to
match whatever midge currently does. The one exception is §6.4 point 3 below, where
the *original spec text itself* was wrong (verifiably: it contradicted this
format's own recovery model) and has been corrected to match what's actually
correct, which happens to be what midge already does.

Most of the format is confirmed to match exactly (frame/TLV layouts, block and
footer encodings, journal framing, CRC algorithm choices, naming schemes).

**Resolved (confirmed) TODOs**, now stated as settled facts in the docs instead of
`TODO: verify`:
- `format/manifest.md` §2 — `FORMAT`(3)/SST `format_version`(4)/WAL payload
  `version`(1) confirmed as three genuinely independent constants with no
  cross-check code anywhere.
- `format/manifest.md` §4.3 — the `ManifestEdit` JSON tagged-union shape is
  confirmed externally-tagged (`{"VariantName": <fields>}`); the bare/legacy edit
  shape is confirmed read-only, never written by a current writer.
- `format/manifest.md` §5.3 / §9 — "`id 0` is the default CF" is confirmed to be an
  application-layer convention only, not enforced by the manifest format itself; the
  legacy `DropColumnFamily` variant is confirmed never written by a current writer.
- `format/wal.md` §6.4/§7 — `writer_epoch == 0` confirmed as a deliberate
  fencing-disabled sentinel, not an incidental default.
- `format/wal.md` §5.3/§7 — the legacy split-marker transaction encoding is
  confirmed **still actively written** (not merely legacy-read) — used as the
  large-transaction "spill" fallback when a transaction is too big for one atomic
  `TxnBatch` frame.
- `format/wal.md` §1.1 — `wal_{segment:06}.log` confirmed never *written* by any
  current code path; it's decode-side-only.
- `format/sst.md` §3.3/§9 — `RESTART_INTERVAL` confirmed fully vestigial, zero
  reader-visible wire effect.
- `format/sst.md` §7.1 — the footer `Corruption`-vs-`CompatibilityError` decision
  tree fixed earlier this version is confirmed to match the reference
  implementation's actual branching exactly.
- `format/lease.md` §1/§5.3/§7 — no OS-level lock is taken (confirmed dead code from
  a prior design); the release sentinel timestamp is confirmed to be exactly
  `1970-01-01T00:00:00Z`; duplicate-field last-occurrence-wins is confirmed for the
  reference implementation.
- **The lease `epoch` ↔ WAL `writer_epoch` relationship** (the single biggest open
  cross-document gap identified earlier this version) is resolved: they are the
  same value by construction — `writer_epoch` is seeded verbatim from the lease's
  granted `epoch` once, at startup, before WAL replay runs, and never reassigned
  afterward. Documented in both `format/lease.md` §8 and `format/wal.md` §7.

**Spec decided, rather than left open** — `format/sst.md` §3.2/§3.4/§9: `Merge`
(`entry_type = 3`) and a value-bearing `Delete` (`value_len > 0`) are now both
explicitly **RESERVED** — a conforming writer MUST NOT emit either until a future
revision defines real semantics for them. Both are confirmed already dead in midge
(no writer path produces either), so this ruling requires no change there; it just
makes midge's existing behavior a spec requirement instead of an accident.

**Spec correction** (the spec's own text was wrong, not midge):
- `format/wal.md` §6.4 point 3 — this document previously stated that epoch-mixing
  within one physical WAL file is corruption. **That was incorrect and has been
  reversed.** Confirmed against midge: its active WAL file is reopened in append
  mode across a writer failover, never rotated at the epoch boundary, so an
  ordinary crash-then-failover cycle routinely and correctly produces one `wal.log`
  (and potentially a sealed segment) spanning more than one `writer_epoch`. The
  format's staleness rule (point 2) is the sole, sufficient mechanism for resolving
  this — a reader that rejected epoch-mixing as corruption, per the old text, would
  break on completely ordinary recovery. The narrower, genuinely-correct
  requirement — that an object *uploaded* to the cloud under an epoch-scoped key
  must itself be single-epoch — has been added to §1.2 instead, which is what
  midge's two epoch-mixing checks actually validate (both are cloud-upload-path-only
  in midge, never on the local recovery path, which is now understood to be
  correct, not a gap).

**Normative claim about "both implementations" corrected to a plain fact**:
- `format/lease.md` §3.3 — the mutation lock file's separator is `=`
  (`field=value`), not `": "` as previously stated; it does not share the leader
  record's line style. (This is a wire-format description fix, not a design
  decision — there's no "should" here, just what bytes are on disk.)

**Known gaps in midge, confirmed by this audit — spec upheld, midge needs fixing**
(most consequential first):
- `format/lease.md` §4 step 4 / `format/wal.md` §7 — **the most significant
  finding of this audit.** The spec requires lease acquisition to accept a
  caller-supplied minimum epoch, computed by the recovering engine from its own
  WAL/manifest history, and use it as a floor. This is what guarantees a newly
  granted epoch is actually higher than anything already durable in this engine's
  own on-disk state — the property fencing depends on. Without it, an acquisition
  that only computes `current_lease_epoch + 1` can grant an epoch that is *not*
  higher than epochs already present in the WAL (e.g. if the leader record and the
  WAL directory ever diverge independently — restore from backup, manual recovery,
  directory migration) — silently reintroducing the stale-writer ambiguity fencing
  exists to prevent. **Confirmed: midge does not implement this.** Its
  lease-acquisition API takes no epoch-floor parameter, and lease acquisition
  completes in full *before* WAL replay runs at all, so `max_epoch_seen` (which
  midge does compute during replay) is never available to feed back in. See
  `format/lease.md` §4 step 4 for the recommended fix (a lightweight preliminary
  scan before/during acquisition).
- `format/lease.md` §7 — **safety-relevant**: midge treats an empty or
  field-malformed leader record identically to an *absent* one ("no lease," epoch
  resets to 0) instead of failing closed as `indeterminate` per the spec, weakening
  fencing on a corrupted record. Recommend fixing midge to fail closed as specified.
- `format/lease.md` §6 — midge's hot-path WAL-sync-boundary fencing check
  (`validate_epoch`) compares only `epoch`, not `holder_id`, narrower than §6 item
  1's stated contract (defense-in-depth against a tampered record or a
  non-conforming writer). Recommend extending it to also compare `holder_id`.
- `format/sst.md` §3.3/§9 — the extended-length-block writer-promotion boundary:
  §3.2 requires a writer to promote to the extended form whenever the real
  key-delta length reaches 65,535 bytes, unconditionally. Midge's writer only
  promotes when the value length *also* needs promotion, so a 65,535-byte key delta
  with an ordinary value length is written as the literal, spec-forbidden byte
  pattern `0xFFFF`. This is silently self-consistent only because midge's reader
  also (non-spec-compliantly) requires both fields to match their sentinel — a
  second, correctly-spec-compliant reader would misparse midge's output at this
  boundary. Recommend fixing midge's writer to promote off `key_delta_len` alone.
- `format/manifest.md` §4.3 — `SetCloudCheckpoint`'s `apply_edit` handler advances
  on `>=` rather than the strict `>` used by `BumpWalSeq`/`BumpNextSstSeq` for the
  same kind of monotonic counter. Low severity (doesn't appear to break the
  idempotent-replay guarantee this document actually depends on) but inconsistent;
  flagged as `TODO: verify` intent rather than a required fix, pending confirmation
  either way.

**Not a gap — implementation choice within the spec's contract**:
- `format/lease.md` §5.3 — release is single-attempt best-effort at one midge call
  site but retried indefinitely (in a non-blocking background thread) at another;
  both satisfy the spec's actual requirement ("never blocks or fails shutdown"), so
  no fix needed — documented as a clarifying implementation note, not a divergence.

Remaining genuinely open (git-history/second-implementation questions the current
midge source couldn't answer as of the audit above): what changed at each prior
manifest `FORMAT` version; whether `wal_{segment:06}.log` or split
`DropColumnFamily` were ever historically *written* by a since-removed code path;
and whether a merge-operator concept exists in the C# engine (which would mean the
`Merge` reservation above needs to be lifted in favor of a real specification, not
left standing).

### Cross-checked against the second reference implementation ("pants", C#)
A `cntryl/pants` repository exists alongside `midge`, with its own C#
implementation and an actual midge↔pants compatibility-fixture test harness
(`eng/compat/Pants.CompatibilityHarness`, `test/Pants.Tests/Compatibility/`). Its
lease implementation (`src/Pants/Storage/Internal/MidgeFileLease.cs`) was read
directly and compared against `format/lease.md`, resolving several previously-open
questions and, most importantly, correcting an earlier over-confident conclusion:

- **The caller-supplied-minimum-epoch mechanism (`lease.md` §4 step 4) is
  confirmed real** — the C# engine implements it (`MidgeFileLease.Acquire(root,
  minimumEpoch, ...)`, threaded from a public `minimumWriterEpoch` parameter,
  default `0`, on its top-level `Open(...)` API). This settles that "both known
  implementations expose a caller-supplied minimum" was true as originally
  written — just not for midge. **Correction to last round's fix**: the previously
  recommended fix for midge (an automatic internal WAL-pre-scan before
  acquisition) was speculative and has been walked back to what's actually
  evidenced — add the equivalent parameter to midge's own API surface, matching
  the C# engine's actual design, rather than inventing an internal-scan mechanism
  neither engine is confirmed to implement automatically. Also added an honest
  caveat that even the C# engine's own default (`0`) leaves this unprotected
  unless a caller supplies a real value — this isn't a solved problem in either
  engine, just a gap that's structurally closable in one and not the other.
- **`lease.md` §6** (holder_id fencing gap) — confirmed the C# engine's equivalent
  hot-path check, `EnsureValid()`, already compares both `epoch` and `holder_id`;
  midge's `validate_epoch` doesn't. The recommended fix for midge now cites a real
  existing reference implementation, not a hypothetical stricter design.
- **`lease.md` §7** (corrupted-record handling) — confirmed the C# engine's
  `ReadRecord` correctly fails closed as indeterminate on both an empty file and a
  field-incomplete record; midge doesn't. Same treatment: cites a working
  reference fix, not just "midge should be stricter."
- **Resolved as confirmed cross-implementation format requirements** (previously
  `TODO: verify`), each independently verified in both engines' source:
  `.midge_leader`/`.midge_leader.lock` filenames; the leader record's field
  order/`": "` separator; the lock file's `field=value` separator (distinct from
  the leader record's); and the release sentinel `1970-01-01T00:00:00Z` (identical
  literal string in both engines).
- **`lease.md` §7 duplicate-field handling** — precisely re-attributed: the C#
  engine (previously described only as "one implementation studied") is confirmed
  to reject duplicate fields, where midge (the spec's designated authority here)
  does last-occurrence-wins. Added a design note (not a reversal — this was
  already a considered decision, unlike the epoch-mixing bug) that the C# engine's
  reject-and-fail-closed behavior may actually be more consistent with this
  document's fail-closed philosophy elsewhere, worth weighing if this section is
  revisited.

Also resolved directly from source (no pants involved): `format/value-encoding.md`'s
open compression-threshold question — confirmed as `MIN_COMPRESSION_INPUT_BYTES =
256` bytes in midge, shared by both the WAL and SST compression paths (also
independently confirmed at the same 256-byte value in pants).

### Second spec-was-wrong correction, found via deeper cross-checking
- `format/sst.md` §3.2/§3.3/§9 — **the extended-length-block writer-promotion
  "bug" flagged against midge earlier this version wasn't a bug.** This document's
  rule ("`key_delta_len == 0xFFFF` is never a legitimate literal length; a writer
  must always promote at exactly 65,535 bytes") was itself too strict. Midge's
  reader requires *both* `key_delta_len` and `value_len` to match their sentinel
  before treating a record as extended — a conjunctive check — and midge's writer
  behavior is safely decodable under that real contract. §3.2/§3.3 now state the
  precise minimal writer requirement (promote when either field's real value can't
  fit its inline width, or when both would coincidentally equal their sentinel
  simultaneously) instead of the unnecessarily strict single-field rule. No change
  needed in midge.

### A note on pants (C#) going forward
`pants` was also spot-checked against `wal.md`, `sst.md`, and `manifest.md` this
round, beyond the lease.md cross-check above. Per guidance, its findings are being
weighted lightly — it's an earlier-stage implementation than midge, so its rough
edges (e.g., it doesn't distinguish `Corruption` from `CompatibilityError` on SST
footer failures the way midge does; it doesn't validate several manifest-carried
SST names at replay time; it has no manifest-journal repair-on-next-write
mechanism) aren't being written up as "known gaps to fix" the way midge's are —
they're just where an in-progress implementation currently stands, not confirmed
spec violations worth prescribing a fix for yet. Two things from this round *are*
folded into the docs above because they hold up regardless of pants' maturity: the
corroboration that both engines reopen-and-append across WAL failover (strengthens
the epoch-mixing correction), and the plain factual correction that
`format/manifest.md`'s "C# engine only reads manifest state" claim is no longer
true (pants now has its own manifest writer, per `LocalDiskStore.cs`) — noted in
the doc without elevating pants to equal authority with midge.
