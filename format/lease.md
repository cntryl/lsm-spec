# Single-Writer Lease File Format

Status: draft, extracted from two independent implementations (a Rust engine and a
C# engine) for cross-checking. Sections marked "TODO: verify" reflect points where
the two implementations agree today but the spec author could not find an explicit,
independent design doc confirming the choice was deliberate (as opposed to
implementation-shared incidental behavior).

## 1. Overview

The lease file is the single-writer enforcement mechanism for a local (non-cloud)
LSM-tree database directory. Exactly one process may hold write access to a given
database directory at a time; the lease file is the persistent, on-disk record of
who currently holds that right, and carries a fencing token so that a writer whose
lease has lapsed can be detected and rejected rather than silently corrupting state
by writing concurrently with a newer holder.

This is distinct from, and does not replace, any OS-level advisory file lock
(`flock`/`LockFileEx`): the record described here is a **persistent CAS-via-rename
leader record**, not an OS lock, specifically so that leadership determination works
correctly on filesystems where advisory locks are unreliable or unsupported (e.g.
NFS, SMB/CIFS shared mounts). **Confirmed** against the Rust reference
implementation: no OS-level lock is taken anywhere in the lease acquisition/renewal/
release path today. `flock`/`LockFileEx` wrapper code still exists at the low-level
I/O abstraction layer — a leftover from the previous, replaced design the doc
comments there still describe — but it has zero call sites in the lease code or
anywhere else in the codebase; it is dead code. The leader record is the sole
mechanism in this implementation. This does not settle whether a *future*
implementation is required to also take an OS lock as defense-in-depth — only that
midge today does not.

Both known implementations use this mechanism only for local/filesystem-backed
databases. A cloud-backed deployment has an analogous lease concept implemented
against the cloud storage provider's conditional-write primitives instead of a local
file; that mechanism is out of scope for this document.

### 1.1 File and directory layout

The lease state lives directly in the database's root directory (the same directory
that contains WAL, SST, and manifest state — not a subdirectory), as two files:

- **The leader record**, a small persistent file whose one instance represents the
  current (or most recently known) lease holder. It survives process exit; on
  restart, its presence and content are what a new-would-be-writer inspects to decide
  whether it may take over.
- **The acquisition/mutation lock**, a transient, zero-or-momentarily-one-instance
  file used only to serialize *concurrent attempts to mutate the leader record*
  (acquire, renew, or release). It is created immediately before a mutation and
  removed immediately after, by whichever process created it. Unlike the leader
  record, its presence outside the span of one mutation attempt is itself an anomaly
  (see §5.2).

**Confirmed** as a real cross-implementation requirement, directly verified against
both reference implementations' source (not merely observed on disk): the two
known implementations agree on:

| File | Purpose |
|---|---|
| `.midge_leader` | Leader record (persistent) |
| `.midge_leader.lock` | Acquisition/mutation lock (transient) |

Both files are UTF-8 text (not a binary format — see §2). No wire-format reason
requires these exact names; they are called out here only because both known
implementations use them, so a reader/writer of an existing database directory must
still recognize them under these names to interoperate with data written by either
engine today.

## 2. Terminology

- **Leader record** — the persistent file recording the current lease holder's
  identity, fencing epoch, and last-refreshed timestamp.
- **Holder** (or **holder identity**) — an opaque string that uniquely identifies one
  lease-acquiring process instance. Used only for identity comparison and
  diagnostics; its internal structure is not format-significant (see §3.2).
- **Epoch** (fencing token) — a monotonically increasing, unsigned integer counter.
  Every successful acquisition strictly increases it relative to the previous
  leader record's epoch (or relative to 0, if none existed). It is the sole basis for
  determining whether a given holder's lease has been superseded (§6).
- **Acquire** — the operation of a process becoming the current lease holder,
  producing a new leader record with a higher epoch.
- **Renew** (heartbeat) — the operation of the current holder refreshing the leader
  record's timestamp without changing its epoch, to signal it is still alive.
- **Release** — the operation of the current holder voluntarily giving up the lease,
  recorded by writing a record whose timestamp is set to a value guaranteed to read
  as maximally stale (see §5.3), while — in at least one known implementation —
  leaving the epoch and holder identity fields otherwise unchanged.
- **Fencing** — the property that a process which held the lease at epoch *E* but has
  since had its lease superseded by a later acquisition at epoch *E' > E* must refuse
  to perform further writes, detectable purely from the leader record's contents
  (§6).
- **Stale** (of a leader record) — a record whose last-refresh timestamp is old
  enough, relative to a policy threshold, that a competing process may treat the
  recorded holder as dead and attempt takeover (§6.2). Staleness is a *policy*
  judgment made by whoever reads the record; it does not change the record's bytes.

## 3. Leader record: structural layout

The leader record is a UTF-8 text document, not a fixed-width binary structure. It
consists of one field per line, `\n`-terminated, in the form:

```
{field-name}: {value}
```

### 3.1 Fields

| Field | Type | Description |
|---|---|---|
| `epoch` | unsigned 64-bit integer, decimal ASCII | Fencing token (§2). Strictly increases across successful acquisitions. |
| `holder_id` | string | Opaque identity of the current/most recent holder (§3.2). |
| `acquired_at` | RFC 3339 timestamp, UTC | Time the current epoch's holder last acquired or renewed the lease. This is the sole liveness signal (§5, §6). |

No other fields are defined. There is no magic number, version byte, or checksum
field in the record itself (see §5.4 on corruption handling, which relies on
whole-field parseability rather than a checksum).

**Confirmed as a real cross-implementation format requirement, not incidental
behavior**: both known implementations independently parse fields by scanning each
line on its own — tolerant of any field order on read — but always *write* them in
exactly `epoch`, `holder_id`, `acquired_at` order with a `": "` (colon-space)
separator. Verified directly against both reference implementations' source: the
Rust engine's writer (`format_leader_record`) and the C# engine's writer
(`MidgeFileLease.WriteRecord`) both produce the byte-identical line format
`epoch: {epoch}\nholder_id: {holder_id}\nacquired_at: {acquired_at}\n`. Two
independently-built implementations agreeing on both field order and separator,
despite the reader being order-tolerant either way, is what elevates this from
"one implementation's incidental writer behavior" to an actual interop contract — a
third implementation should reproduce this exact write order/separator to remain
maximally compatible with tooling that might parse the file positionally, even
though a spec-compliant reader must not require it.

### 3.2 Holder identity

Both known implementations construct `holder_id` by concatenating process id,
a locally-unique instance/thread discriminator, and host identity, e.g.
`{pid}.{instance_or_guid}@{hostname}`. This exact construction is **implementation
policy, not format** — a conforming reader must treat `holder_id` as an opaque
string used only for equality comparison (does this record still belong to *me*?)
and for operator-facing diagnostics. No parser may assume any internal structure
(e.g. that it contains an `@`, or that the portion before it is numeric).

### 3.3 The acquisition/mutation lock file

The `.lock` file is also UTF-8 text, using a `{field}={value}`-style line format
(not reusing the leader record's three fields — and, **confirmed identical across
both reference implementations, not the leader record's `": "`-separated style
either**: the Rust engine writes
`holder_id={holder_id}\nowner_token={owner_token}\ncreated_at={created_at}\n`, and
the C# engine's `AcquireMutationLock` independently produces the exact same
`holder_id={...}\nowner_token={...}\ncreated_at={...}\n` shape — a bare `=` with no
space, distinct from the leader record's `": "` separator in §3.1. Two
independently-built implementations agreeing on this is strong evidence it's the
deliberate shared contract, not one engine's incidental choice). Observed content:

| Field | Type | Description |
|---|---|---|
| `holder_id` | string | Identity of whoever is currently mutating the leader record. |
| `owner_token` | string (UUID) | Random token minted fresh per lock acquisition, used to detect and refuse removing a lock that a different (later) acquisition attempt now owns. |
| `created_at` | RFC 3339 timestamp, UTC | When this lock instance was created. Diagnostic only — **TODO: verify** whether any implementation uses it to break a stuck lock automatically; the reference implementation explicitly does not (see §5.2). |

The lock's *existence* is created with a filesystem primitive that fails if the file
already exists (`create_new`/O_EXCL semantics) — this exclusivity, not the file's
content, is what actually serializes concurrent mutators. The content exists for
diagnostics and for the `owner_token`-checked removal in §5.2, not to communicate
lease state to unrelated readers.

## 4. Acquisition semantics

Acquisition is a compare-and-swap performed via a lock-file-guarded read of the
current leader record followed by a durable write of a new one:

1. Create the mutation lock file, exclusively (fails if another process is
   concurrently acquiring/renewing/releasing).
2. Read the current leader record, if any.
3. If a current record exists and is not already owned by this process:
   - Parse its `acquired_at` timestamp. An unparseable timestamp, or one strictly in
     the future relative to the reader's clock, makes ownership status
     **indeterminate** — the acquisition attempt must fail closed rather than assume
     either liveness or staleness (§6.3).
   - Compute the record's age. If the age is below the staleness threshold (policy;
     §7), the acquisition fails: the existing holder is presumed live.
   - If the age is at or beyond the staleness threshold, the acquisition may proceed
     (a "takeover" of a presumed-dead holder).
4. Compute the new epoch as `max(current epoch, any caller-supplied minimum) + 1`.
   **A conforming acquisition implementation MUST support this caller-supplied
   minimum, and a recovering engine MUST supply it, computed from its own
   WAL/manifest recovery pass, before treating itself as open for new writes.**
   This is not an optional convenience parameter: it is what guarantees the epoch
   this acquisition grants is higher than every epoch value that could already be
   durable in this engine's own on-disk state (WAL records the engine is about to
   replay, or has already replayed) — the single property fencing (§6) depends on.
   Without it, an acquisition that only computes `current_lease_epoch + 1` can grant
   an epoch that is *not* actually higher than epochs already present in this
   engine's own WAL — e.g. if the leader record was lost/reset independently of the
   WAL directory (restore from an old backup, manual recovery, directory migration)
   — silently reintroducing exactly the kind of stale-writer ambiguity fencing
   exists to prevent, on the *next* recovery pass rather than this one. Reaching
   `u64::MAX` fails the acquisition (**epoch exhausted**) rather than wrapping.

   **Confirmed real and deliberate, not a spec artifact: the C# reference
   implementation implements exactly this.** `MidgeFileLease.Acquire(root,
   minimumEpoch, ...)` computes `Math.Max(current?.Epoch ?? 0, minimumEpoch) + 1` —
   directly matching this step's formula — and `minimumEpoch` is threaded in from a
   public `minimumWriterEpoch` parameter on the C# engine's top-level
   `LocalDiskStore.Open(...)` API. This settles the earlier open question of
   whether "both known implementations expose a caller-supplied minimum" was ever
   true: it was, for the C# engine. **It is not true for midge.**

   **Known gap in the reference implementation ("midge") — recommended fix:**
   confirmed midge's `LeaderStore`/`PrimaryLease` acquisition APIs take no
   equivalent epoch-floor parameter at all — not even the plumbing exists, let
   alone anything that computes a value for it. Its lease acquisition also
   completes in full *before* WAL replay runs, and WAL replay's own
   `max_epoch_seen` output (`wal.md` §6.4 point 4) is never fed back into — or even
   reachable from — acquisition. Recommended fix, matching the C# engine's actual
   design: add an equivalent minimum-epoch parameter to midge's own
   lease-acquisition / engine-open API surface, so a caller with out-of-band
   knowledge of a higher epoch (an operator running disaster recovery, a
   multi-replica coordinator, a migration tool) has a way to supply it — parity
   with what the C# engine already exposes, not a novel design.

   **One honest caveat, so this isn't overclaimed as "solved by the C# engine":**
   the C# engine's own default is `minimumWriterEpoch = 0`, and its `Open()` also
   acquires the lease *before* loading the manifest — so out of the box, with no
   caller passing a non-default value, the C# engine is exposed to the exact same
   hazard described above; it merely *can* be protected against, by a caller that
   knows to use the parameter, in a way midge currently cannot be protected against
   at all. Neither engine automatically closes this gap on its own; midge is simply
   missing even the option. (The `u64::MAX` exhaustion behavior above is confirmed
   correct in both engines — midge's `checked_add(1)` and the C# engine's explicit
   `== ulong.MaxValue` check both fail closed rather than wrapping.)
5. Write the new record — `{epoch: new_epoch, holder_id: this_process, acquired_at:
   now}` — via write-to-temp-file + fsync + atomic rename over the leader record path
   + fsync the containing directory. (This is the same staged-write pattern used
   elsewhere in the engine for other metadata files; see the WAL spec's discussion of
   atomic replacement for the general pattern.)
6. Re-read the leader record and confirm it now shows this process's `holder_id` at
   the expected new epoch. If it does not, the acquisition lost a race and fails.
   (In principle this should be unreachable given the exclusive mutation lock in
   step 1, but both known implementations treat it as a real failure mode and check
   it explicitly — **TODO: verify** exactly which unguarded external-writer scenario
   this check is meant to catch.)
7. Release the mutation lock (remove the lock file), only if it is still the same
   lock instance this process created (checked via `owner_token`, §3.3 / §5.2).

A successful acquisition strictly increases the epoch relative to every record that
has ever existed at this path (subject to the caller-supplied minimum in step 4).
This is the property fencing depends on (§6).

## 5. Renewal (heartbeat) and release semantics

### 5.1 Renewal

A held lease's holder periodically refreshes the leader record to prove liveness.
Renewal:

1. Acquires the mutation lock (same exclusivity as acquisition).
2. Re-reads the current record and verifies it still shows this process's
   `holder_id` **and** the epoch it was granted at acquisition time.
   - If either does not match, the lease has already been superseded: renewal fails,
     and the holder must treat itself as fenced (§6) — it must not attempt to
     "reclaim" the lease by writing over the newer holder's record.
3. If ownership still matches, writes a new record with the **same** `epoch` and
   `holder_id`, and `acquired_at` set to the current time — via the same
   staged-write pattern as acquisition.
4. Re-verifies the write took effect under the same holder/epoch it wrote, failing
   renewal (and self-fencing) if not.
5. Releases the mutation lock.

Only the timestamp changes on a successful renewal. The epoch and holder identity
are format-invariant across every renewal within one lease lifetime — a competing
reader can therefore always tell "same lease, still alive" (matching epoch, newer
timestamp) apart from "lease changed hands" (different epoch and/or holder).

**Known gap in the reference implementation ("midge") — recommended fix:** the
fencing check §6 item 1 requires comparing *both* epoch and `holder_id` — a
`holder_id` mismatch alone, even at an unchanged epoch, must mean superseded. Under
this format's own epoch-CAS acquisition protocol (§4), two different holders should
never legitimately share one epoch, so epoch-only comparison is not *known* to be
exploitable through §4 alone — but the whole point of also checking `holder_id` is
defense against exactly the cases §4 doesn't cover: a directly-tampered or
externally-rewritten leader record, a bug in a *different*, non-conforming writer
sharing the directory, or a defect in another conforming implementation's own CAS
logic. Relying solely on epoch equality discards that defense-in-depth for no
benefit — the comparison is no more expensive to also check `holder_id`. Confirmed
against the Rust reference implementation: its runtime fencing check actually
invoked at WAL-sync boundaries (`LeaderStore::validate_epoch`, checked before every
durable WAL sync) compares **only** epoch, never `holder_id` — narrower than §6
item 1 requires. (Renewal's own self-check, step 2 above, does compare both
correctly; it's specifically this separate hot-path sync-boundary check that needs
the same treatment.) Recommended fix: extend `validate_epoch` (or add an equivalent
check on the same hot path) to also compare the in-memory `holder_id` against the
live leader record's, matching what §6 item 1 requires — and matching the C#
reference implementation's own equivalent hot-path check, `EnsureValid()`, which
already compares both (`current?.Epoch != Epoch || current.HolderId != _holderId`).
This isn't a hypothetical stricter design being proposed here for the first time;
it's what the other reference implementation already does.

*How often* renewal happens (the heartbeat interval) is pure timing policy, out of
scope for this document — see §8.

### 5.2 The mutation lock is never broken automatically

Neither known implementation ever removes a pre-existing, still-present mutation
lock file to "unstick" a crashed mutator. A lock file is removed only by the same
logical acquisition attempt that created it (verified via `owner_token`), and if a
lock file already exists, every operation that needs the mutation lock fails
immediately as contended, rather than waiting, retrying with backoff, or assuming
the lock is abandoned. Rationale documented in the reference implementation: an
in-flight rename on some filesystems (notably NFS/SMB) cannot be reliably
cancelled, so it is never safe to assume a lock-holder that appears to have crashed
has *actually* stopped mutating the target file. Recovering from a stuck mutation
lock is an explicit operator action (out of band, not a format-level or automatic
engine behavior).

### 5.3 Release

Release is a voluntary, best-effort downgrade of a still-owned record so that a
future acquirer's staleness check (§4 step 3) succeeds immediately rather than
waiting out the full staleness policy window. Observed technique: the current
holder, while still holding the mutation lock and confirming it is still the record
owner (same check as renewal), rewrites the record with the epoch and holder
identity **unchanged** but `acquired_at` forced to an ancient/minimum timestamp
(observed value: the Unix epoch, `1970-01-01T00:00:00Z`) — guaranteeing any age
computation against it exceeds the staleness threshold.

**Confirmed as the intended cross-engine format contract**, not one
implementation's convenience: the C# reference implementation's `Dispose()`
independently produces the exact same sentinel, `AcquiredAt = "1970-01-01T00:00:00Z"`
(down to the literal string), on the same unchanged-epoch/holder_id rewrite. Two
independently-built implementations landing on the identical sentinel string,
rather than merely "some ancient timestamp," is strong evidence this specific
value is deliberate rather than incidental — a third implementation should use
this exact string, not just any sufficiently-old timestamp, for maximal
byte-level interop with tooling that might compare records literally. Deleting the
record instead was not observed in either implementation.

Release does not increase the epoch. The next successful acquirer still computes
`epoch = previous_epoch + 1`.

Release is explicitly best-effort: if it cannot complete (I/O error, lock
contention), the holder proceeds with shutdown anyway. The record is left as the
holder's last-renewed state, and a future acquirer takes over once the ordinary
staleness window elapses.

**Implementation note:** "best-effort" here describes the *contract* (a failed
release must never block shutdown or be treated as a fatal error) — it doesn't
mandate exactly one attempt. The Rust reference implementation has two release call
sites with different retry behavior: a single-attempt best-effort release during
ordinary startup-failure cleanup, and a separate shutdown path that retries release
indefinitely in a detached background thread — so a transient failure doesn't leave
a live-looking record behind, without that retry loop blocking the shutdown that
triggered it. Both satisfy the contract above; a from-scratch implementation is
free to choose either strategy (or another) as long as release failure never blocks
or fails shutdown.

## 6. Fencing semantics

Any process — a would-be new writer, or a diagnostic reader — determines whether a
lease it once held (or observed) is still current purely by re-reading the leader
record and comparing:

1. **Superseded**: the current record's `epoch` no longer equals the epoch this
   process was granted, or its `holder_id` no longer equals this process's own
   identity (even at the same epoch — see the note below). Either mismatch means
   this process's lease is no longer valid, unconditionally, regardless of how
   recent the record's timestamp is. A fenced process must reject all further
   writes; both known implementations treat this as effectively unrecoverable for
   the current lease instance (no "reclaim," only a *new* acquisition can restore
   write access, yielding a strictly higher epoch).
2. **Live vs. stale (for a would-be *acquirer*, not an existing holder)**: a record
   whose `holder_id` is not this process's own is judged live or stale purely by the
   age of `acquired_at` relative to a policy threshold — see §4 step 3 and §8. There
   is no separate "lease expired" flag in the record; expiry is entirely a function
   of (current time − recorded time), recomputed by whoever is asking.
3. **Indeterminate is not "safe to proceed"**: an unparseable or future-dated
   `acquired_at` must be treated as a hard failure of the acquisition/validity check
   — never coerced into either "definitely live" or "definitely stale." This applies
   symmetrically to acquisition (§4) and to an existing holder's self-check (item 1
   partially depends on parsing too, though the epoch/holder_id comparison itself
   does not require timestamp parsing and should be checked independently of it).

The epoch is the only value that gives fencing an unambiguous, clock-independent
answer ("has this specific lease been superseded" — item 1). The timestamp
comparison (item 2) is inherently clock-dependent and only ever used to gate
*acquisition* of a new lease over a *presumed-dead* holder, never to invalidate a
currently-valid one out from under it.

**TODO: verify** — whether a holder whose own epoch still matches, but whose
`acquired_at` has (through some fault, e.g. an external process or clock issue)
drifted to look stale to *other* readers, must proactively distrust its own lease
even though the epoch/holder_id check (item 1) still passes. Neither implementation
appeared to add this as a self-check; the holder relies on renewal succeeding or
failing, not on re-deriving its own staleness.

## 7. Corruption and validation rules

- A leader record file that exists but is empty, or whose bytes are not valid UTF-8,
  must be treated as **indeterminate**, not as "no lease" and not as "definitely
  stale" — see §6 item 3.
- A record missing any of the three required fields (`epoch`, `holder_id`,
  `acquired_at`), or with an `epoch` value that does not parse as a non-negative
  integer within the field's width, must likewise be treated as indeterminate.

  **Known gap in the reference implementation ("midge") — safety-relevant, fix
  needed:** midge does not follow either rule above as written. An **empty** leader
  record file, and a **present-but-field-incomplete-or-malformed** record, both
  return the same result as an *absent* file ("no lease," i.e. an effective epoch
  baseline of 0 for the next acquirer) rather than failing closed as indeterminate —
  a corrupted-but-present record silently resets fencing state instead of blocking
  acquisition. Separately, a **non-UTF-8** record does fail closed in midge, but as
  a generic I/O error rather than the specific indeterminate/error classification
  this document describes. A from-scratch implementation that follows this
  document's literal text (fail closed as indeterminate on all three conditions)
  will be *stricter and safer* than the current reference implementation, not
  merely different from it — this reads as a bug in midge relative to its own
  extracted spec, worth fixing there rather than relaxing the rule here. The C#
  reference implementation gets this right, and can serve as the fix reference:
  `MidgeFileLease.ReadRecord` throws its indeterminate exception both when the file
  is empty (no lines parse into any field) and when any of the three required
  fields is missing after parsing — it never conflates either case with "absent
  file." Midge should be brought in line with that behavior.
- A record containing a **duplicate** field (the same field name on two lines) is
  **not** corruption: the reference parser scans lines in order and simply
  overwrites each field's value as it re-encounters that field name, so the
  **last** occurrence of a given field in the file wins. A reader must reproduce
  this exact behavior (last-occurrence-wins), not reject duplicates. This is a
  standing decision, not an open question: an earlier draft of this document
  flagged a cross-implementation divergence here — the C# reference
  implementation's `MidgeFileLease.ReadRecord` does reject a duplicate field
  (throwing its indeterminate exception), directly confirmed against its source —
  and this document deliberately designates the Rust engine's last-occurrence-wins
  behavior as authoritative for the shared format, not the C# engine's rejection.
  *(Worth weighing if this decision is ever revisited: no conforming writer can
  produce a duplicate field in the first place — every write replaces the whole
  file atomically in one shot — so a duplicate's mere presence is itself evidence
  of external tampering or a non-conforming writer, the same category of "trust
  nothing, fail closed" situation §6 item 3 and this section's other rules already
  handle by failing closed rather than silently picking an interpretation. The C#
  engine's reject-as-indeterminate behavior is arguably more consistent with that
  pattern than the standing last-occurrence-wins rule is. Restated here as a
  design note for whoever next revisits this section, not as a reversal of the
  standing decision above.)*
- There is no checksum over the leader record. Corruption/tampering that produces
  syntactically valid-but-wrong field values (e.g. bytes flipped inside a still
  parseable ASCII decimal `epoch`) is **not detectable** by this format as
  described. **TODO: verify** whether this is an accepted gap (leader record is
  small, locally written, and rewritten wholesale on every mutation, so silent
  bit-rot is considered out of the practical threat model) or an omission to be
  closed in a future format revision.
- The absence of a leader record file at all is not corruption — it is the
  well-defined "no one has ever held this lease" state, equivalent to a
  hypothetical epoch-0 record for the purposes of the next acquisition's epoch
  computation (§4 step 4).
- A present-but-empty mutation **lock** file, or one whose `owner_token` cannot be
  parsed, must be treated the same as any other "lock is held by someone" case for
  the purposes of exclusivity (fail the mutation attempt as contended) — it must
  never be treated as vacant just because its content didn't parse.

## 8. Format vs. policy — what's in scope here

The following are explicitly **behavioral/policy**, not part of the on-disk format
contract, and are not required for wire compatibility between two engines that both
correctly implement §3–§7:

- **Heartbeat interval** (how often a held lease's holder renews): pure timing
  policy. The *shape* of what renewal writes (§5.1: same epoch/holder, new
  timestamp) is in scope; the cadence is not.
- **Staleness/TTL threshold** (how old `acquired_at` must be before a competing
  acquirer may treat a holder as dead) and any **clock-skew tolerance** added on top
  of it: policy, chosen independently by each deployment/engine build, and expected
  to differ between implementations without breaking interoperability — a
  conforming reader only needs to know *how* to compute age from the timestamp
  (§6 item 2), not what threshold to apply. (Observed reference values — a
  30-second TTL with a 15-second default skew allowance — are illustrative
  defaults, not format constants.)
- **Whether/how a process notifies its own application code of lease loss**
  (callback-on-fenced, polling a health method, etc.): purely an engine API
  ergonomics concern, invisible on disk.
- **Whether an OS-level advisory lock is taken in addition to the leader record**:
  policy/defense-in-depth choice; the leader record alone is what this document
  specifies, and a conforming implementation is not required to also take an OS
  lock (see §1 TODO).
- **Exact holder-identity string construction** (§3.2): policy/diagnostic
  convenience. Only its role as an opaque, comparison-only value is in scope.
- **Operator recovery procedure for a stuck mutation lock** (§5.2): operational
  runbook policy, not format — the format only specifies that automatic removal
  must never happen, not what an operator should do about it.

Ambiguous / needs a decision when writing the canonical spec:

- **Confirmed** — `.midge_leader` and `.midge_leader.lock` are used verbatim by
  both reference implementations (the C# engine's `MidgeFileLease` constructs both
  paths with these exact literal names), not merely the Rust engine's convention;
  see §1.1. Given cross-implementation agreement, treat these as a required format
  detail for interop, not an arbitrary convention a third implementation is free to
  rename.
- **Confirmed** — Release forcing `acquired_at` to the exact sentinel
  `1970-01-01T00:00:00Z` (rather than deleting the record, or a dedicated
  "released" marker) is the intended shared contract: both reference
  implementations independently produce the identical sentinel string; see §5.3.
- **TODO: verify** — Whether the missing checksum over the leader record is an
  accepted gap or an open format issue; see §7.
- **Confirmed** — No implementation requires an OS-level lock as defense-in-depth
  alongside the leader record; midge takes none (dead code remains from a prior
  design). See §1.
- **Confirmed** — The relationship between this document's `epoch` (§2) and the
  WAL's `writer_epoch` (`wal.md` §2, TLV tag 10, §6.4): in the Rust reference
  implementation they are the same value by construction — a successful lease
  acquisition's `epoch` is copied verbatim, once, into `writer_epoch` before WAL
  replay runs, and is never reassigned afterward. This is a one-time seed, not an
  ongoing enforced identity and not an independent counter — see `wal.md` §7 for
  the full trace and its consequence for the caller-supplied-minimum claim in §4
  step 4 above.
