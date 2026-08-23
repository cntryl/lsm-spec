# Single-Writer Lease File Format

Status: draft. Sections marked "TODO: verify" record points that are not yet settled;
they are open questions for this specification, not statements about any particular
implementation. Requirements are expressed with must / must not / required / should /
may in the sense of RFC 2119, whether or not they appear in capitals.

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
NFS, SMB/CIFS shared mounts).
The leader record is the sole mechanism this document specifies. An implementation
may take an OS-level lock as additional defense-in-depth, but must not rely on one
for correctness, and must not require one of its peers; see §8.

This mechanism applies to local, filesystem-backed databases. A cloud-backed
deployment has an analogous lease concept built on the object store's
conditional-write primitives instead of a local file; that mechanism is out of scope
for this document.

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

The two files are named:

| File | Purpose |
|---|---|
| `.midge_leader` | Leader record (persistent) |
| `.midge_leader.lock` | Acquisition/mutation lock (transient) |

Both names are fixed literals. They carry no meaning beyond identifying these files.

Both files are UTF-8 text, not a binary format (§3). Nothing in the encoding itself
depends on these particular names, but because a lease is only meaningful when every
process sharing a directory looks for it in the same place, the names are part of the
interop contract: an implementation that renames them cannot interoperate with data
written by either existing engine. Use them verbatim (§8).

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
  as maximally stale (see §5.3), leaving the epoch and holder identity unchanged.
- **Fencing** — the property that a process which held the lease at epoch *E* but has
  since had its lease superseded by a later acquisition at epoch *E' > E* must refuse
  to perform further writes, detectable purely from the leader record's contents
  (§6).
- **Stale** (of a leader record) — a record whose last-refresh timestamp is old
  enough, relative to a policy threshold, that a competing process may treat the
  recorded holder as dead and attempt takeover (§6 item 2). Staleness is a *policy*
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
field in the record itself (see §7 on corruption handling, which relies on
whole-field parseability rather than a checksum).

**Reading is order-tolerant; writing is fixed.** A reader must parse each line
independently and accept the three fields in any order. A writer must emit them in
exactly this order, with a `": "` (colon-space) separator:

```
epoch: {epoch}\nholder_id: {holder_id}\nacquired_at: {acquired_at}\n
```

The asymmetry is deliberate. Fixing the write order keeps the file byte-comparable
and safe for tooling that parses it positionally, while the tolerant read rule keeps
a conforming reader from rejecting a record that is otherwise perfectly valid.

### 3.2 Holder identity

`holder_id` must uniquely identify one lease-acquiring process instance, and must be
stable for that instance's lifetime. A conventional construction concatenates a
process id, a locally unique instance discriminator, and a host identity — for
example `{pid}.{instance_or_guid}@{hostname}` — but the construction is
**implementation policy, not format**.

A conforming reader must treat `holder_id` as an opaque string, used only for
equality comparison ("does this record still belong to me?") and for operator-facing
diagnostics. No parser may assume any internal structure: not that it contains an
`@`, not that any portion of it is numeric, not that it is bounded in length.

### 3.3 The acquisition/mutation lock file

The `.lock` file is also UTF-8 text, but shares neither the leader record's fields
nor its separator. It uses `{field}={value}` lines with a bare `=` and no
surrounding space, distinct from the leader record's `": "` (§3.1):

```
holder_id={holder_id}\nowner_token={owner_token}\ncreated_at={created_at}\n
```

Fields:

| Field | Type | Description |
|---|---|---|
| `holder_id` | string | Identity of whoever is currently mutating the leader record. |
| `owner_token` | string (UUID) | Random token minted fresh per lock acquisition, used to detect and refuse removing a lock that a different (later) acquisition attempt now owns. |
| `created_at` | RFC 3339 timestamp, UTC | When this lock instance was created. Diagnostic only: it must not be used to age out or break a stuck lock, which §5.2 forbids outright. |

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
     either liveness or staleness (§6 item 3).
   - Compute the record's age. If the age is below the staleness threshold (policy;
     §8), the acquisition fails: the existing holder is presumed live.
   - If the age is at or beyond the staleness threshold, the acquisition may proceed
     (a "takeover" of a presumed-dead holder).
4. Compute the new epoch as `max(current epoch, caller-supplied minimum) + 1`.
   **A conforming acquisition implementation MUST accept a caller-supplied minimum
   epoch, and a recovering engine MUST supply one, computed from its own WAL and
   manifest recovery pass, before treating itself as open for new writes.**

   This is not an optional convenience parameter. It is what guarantees the granted
   epoch exceeds every epoch that could already be durable in this engine's own
   on-disk state — the records it is about to replay, or has just replayed — which is
   the single property fencing (§6) rests on. An acquisition that computes only
   `current_lease_epoch + 1` can grant an epoch that is *not* higher than epochs
   already present in the WAL whenever the leader record and the WAL have diverged:
   restore from an older backup, manual recovery, or directory migration. The result
   is a silent reintroduction of exactly the stale-writer ambiguity fencing exists to
   prevent, detected on some later recovery pass rather than at acquisition.

   Reaching `u64::MAX` fails the acquisition (**epoch exhausted**) rather than
   wrapping.

   A caller with out-of-band knowledge of a higher epoch — an operator running
   disaster recovery, a multi-replica coordinator, a migration tool — supplies it
   through the same parameter. Note that a default of `0` satisfies the letter of
   this step while providing none of its protection: the floor is only as good as the
   value a caller computes for it.
5. Write the new record — `{epoch: new_epoch, holder_id: this_process, acquired_at:
   now}` — by writing to a temporary file, fsyncing it, atomically renaming it over
   the leader record path, then fsyncing the containing directory. This is the same
   staged-write pattern the manifest uses for its snapshot and mirror files
   (`manifest.md` §6.3).
6. Re-read the leader record and confirm it shows this process's `holder_id` at the
   expected new epoch. If it does not, the acquisition lost a race and fails. The
   exclusive mutation lock from step 1 should make this unreachable between
   conforming processes; it is checked anyway as a guard against a non-conforming or
   external writer touching the record. **TODO: verify** which such scenario this is
   specifically intended to catch.
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

**The fencing check applies on every write path, not only at renewal.** §6 item 1
requires comparing *both* `epoch` and `holder_id`, and that comparison must be made
wherever a holder acts on the assumption that it still holds the lease — including
the hot path immediately before each durable WAL sync, not only inside renewal.

A `holder_id` mismatch at an unchanged epoch must be treated as superseded. Under
the epoch-CAS protocol of §4, two holders should never legitimately share one epoch,
so an epoch-only comparison is not known to be exploitable by conforming writers
alone. The `holder_id` comparison exists for what §4 does not cover: a directly
tampered or externally rewritten record, a non-conforming writer sharing the
directory, or a defect in another implementation's CAS logic. It costs nothing beyond
the comparison already being made, so an implementation must not omit it.

*How often* renewal happens (the heartbeat interval) is pure timing policy, out of
scope for this document — see §8.

### 5.2 The mutation lock is never broken automatically

A pre-existing mutation lock file must never be removed to "unstick" a crashed
mutator. A lock file is removed only by the same acquisition attempt that created it,
verified via `owner_token`. If a lock file already exists, every operation needing the
mutation lock fails immediately as contended — it must not wait, retry with backoff,
or assume the lock is abandoned.

The reason is that an in-flight rename cannot be reliably cancelled on some
filesystems, notably NFS and SMB. It is therefore never safe to assume that a
lock holder which *appears* to have crashed has actually stopped mutating the target
file. Recovering from a stuck mutation lock is an out-of-band operator action, not an
automatic engine behavior.

### 5.3 Release

Release is a voluntary downgrade of a still-owned record, so that a future
acquirer's staleness check (§4 step 3) succeeds immediately instead of waiting out
the full staleness window.

The releasing holder, still holding the mutation lock and still confirmed as the
record owner (the same check renewal makes), rewrites the record with `epoch` and
`holder_id` **unchanged** and `acquired_at` forced to the sentinel

```
1970-01-01T00:00:00Z
```

so that any age computation against it exceeds any staleness threshold. This exact
literal must be used, not merely some sufficiently old timestamp, so that records
remain byte-comparable for tooling that inspects them literally. Releasing by
deleting the record is not permitted: the record's absence means "no lease has ever
been held here" (§7), which is a different state and would reset the epoch baseline
for the next acquirer.

Release does not increase the epoch. The next successful acquirer still computes
`epoch = previous_epoch + 1`.

**Release is best-effort.** If it cannot complete — an I/O error, lock contention —
the holder proceeds with shutdown anyway. The record is left in its last-renewed
state, and a future acquirer takes over once the ordinary staleness window elapses.
This constrains the *outcome*, not the number of attempts: a failed release must
never block shutdown or be reported as fatal, but an implementation may retry once,
retry indefinitely on a detached thread, or not retry at all.

## 6. Fencing semantics

Any process — a would-be new writer, or a diagnostic reader — determines whether a
lease it once held (or observed) is still current purely by re-reading the leader
record and comparing:

1. **Superseded**: the current record's `epoch` no longer equals the epoch this
   process was granted, or its `holder_id` no longer equals this process's own
   identity (even at the same epoch — see the note below). Either mismatch means
   this process's lease is no longer valid, unconditionally, regardless of how
   recent the record's timestamp is. A fenced process must reject all further
   writes. This is unrecoverable for the current lease instance: there is no
   "reclaim" operation, and only a *new* acquisition — yielding a strictly higher
   epoch — can restore write access.
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

**TODO: verify** — Whether a holder whose epoch and `holder_id` still match, but
whose `acquired_at` has drifted to look stale to *other* readers (through an external
process or a clock fault), must proactively distrust its own lease. As specified, it
need not: a holder learns it has been superseded from renewal failing, not from
re-deriving its own staleness.

## 7. Corruption and validation rules

- A leader record file that exists but is empty, or whose bytes are not valid UTF-8,
  must be treated as **indeterminate**, not as "no lease" and not as "definitely
  stale" — see §6 item 3.
- A record missing any of the three required fields (`epoch`, `holder_id`,
  `acquired_at`), or with an `epoch` value that does not parse as a non-negative
  integer within the field's width, must likewise be treated as indeterminate.

  These three conditions — empty, non-UTF-8, and field-incomplete — must each fail
  closed as indeterminate, and in particular must **not** be conflated with an
  *absent* file. An absent file means "no lease has ever been held here" and resets
  the epoch baseline to 0 for the next acquirer (see below); a present-but-corrupt
  record means the true baseline is unknown, and treating it as 0 would silently
  discard fencing state at exactly the moment it matters most.
- A record containing a **duplicate** field (the same field name on two lines) is
  **not** corruption. A reader scans lines in order and overwrites each field as it
  re-encounters that name, so the **last** occurrence wins. A reader must reproduce
  this behavior rather than rejecting duplicates.

  *(Design note, for whoever revisits this section: no conforming writer can produce
  a duplicate field, since every write replaces the whole file atomically. A
  duplicate's presence is therefore itself evidence of tampering or a non-conforming
  writer — the same category of situation §6 item 3 and the rules above handle by
  failing closed. Rejecting duplicates as indeterminate would be more consistent
  with that philosophy than last-occurrence-wins is. The rule above stands; this is
  recorded as a considered trade-off, not an open question.)*
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
  (§6 item 2), not what threshold to apply. A 30-second TTL with a 15-second
  clock-skew allowance is a reasonable starting point, but these are illustrative
  values, not format constants.
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

**Settled:**

- The filenames `.midge_leader` and `.midge_leader.lock` (§1.1) are required for
  interop, not a renameable convention: a lease only works if every process
  sharing a directory looks in the same place.
- Release forces `acquired_at` to the exact sentinel `1970-01-01T00:00:00Z` rather
  than deleting the record or writing a dedicated "released" marker; see §5.3.
- An OS-level advisory lock is not required alongside the leader record; see §1.
- This document's `epoch` (§2) and the WAL's `writer_epoch` (`wal.md` §2, TLV
  tag 10, §6.4) are the same value by construction. A successful acquisition's
  `epoch` is copied once into `writer_epoch`, before WAL replay runs, and is never
  reassigned. It is a one-time seed, not an ongoing enforced identity and not an
  independent counter; see `wal.md` §7 for the consequences for §4 step 4 above.

**Open questions:**

- **TODO: verify** — Whether the absence of a checksum over the leader record (§7)
  is an accepted gap or an omission to close in a future revision.
- **TODO: verify** — Whether a duplicate field should remain last-occurrence-wins or
  become indeterminate; see the design note in §7.
