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
NFS, SMB/CIFS shared mounts). **TODO: verify** — whether an engine is additionally
required to take *any* OS-level lock as a secondary defense-in-depth measure, or
whether the leader record is intended to be the sole mechanism; the reference Rust
implementation's history shows OS locks were used previously and were deliberately
replaced by this record-based design.

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

**TODO: verify** exact filenames as a cross-implementation format requirement versus
implementation-specific convenience naming. The two known implementations agree on:

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

**TODO: verify** — field ordering (`epoch`, `holder_id`, `acquired_at`) and the exact
separator (`": "`, i.e. colon-space) as a format requirement versus incidental writer
behavior; both known implementations parse fields by prefix-matching each line
independently and are tolerant of reordering, but always *write* them in this order.

### 3.2 Holder identity

Both known implementations construct `holder_id` by concatenating process id,
a locally-unique instance/thread discriminator, and host identity, e.g.
`{pid}.{instance_or_guid}@{hostname}`. This exact construction is **implementation
policy, not format** — a conforming reader must treat `holder_id` as an opaque
string used only for equality comparison (does this record still belong to *me*?)
and for operator-facing diagnostics. No parser may assume any internal structure
(e.g. that it contains an `@`, or that the portion before it is numeric).

### 3.3 The acquisition/mutation lock file

The `.lock` file is also UTF-8 text, using the same `{field}: {value}`-style line
format (informally; not reusing the leader record's three fields). Observed content:

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
   Both known implementations expose a caller-supplied minimum, used by a recovering
   engine to ensure its own newly issued epoch outranks anything it observed during
   log/manifest replay, not only the epoch in the leader record itself. Reaching
   `u64::MAX` fails the acquisition (**epoch exhausted**) rather than wrapping.
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

**TODO: verify** — whether forcing `acquired_at` to a fixed sentinel-old value
(versus, say, deleting the record entirely, or defining a distinct explicit
"released" marker/field) is the intended cross-engine format contract, or one
implementation's convenient way to piggyback release on the existing staleness
check without adding a new field. Deleting the record was not observed in either
implementation studied.

Release does not increase the epoch. The next successful acquirer still computes
`epoch = previous_epoch + 1`.

Release is explicitly best-effort: if it cannot complete (I/O error, lock
contention), the holder proceeds with shutdown anyway. The record is left as the
holder's last-renewed state, and a future acquirer takes over once the ordinary
staleness window elapses.

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
  stale" — see §6 item 3. (**TODO: verify**: one implementation studied treats a
  present-but-unparseable record file as indeterminate/error; confirm this is
  intended to be universal rather than that implementation's specific choice to fail
  a `TryParse`.)
- A record missing any of the three required fields (`epoch`, `holder_id`,
  `acquired_at`), or with an `epoch` value that does not parse as a non-negative
  integer within the field's width, must likewise be treated as indeterminate.
- A record containing a **duplicate** field (the same field name on two lines) is
  **not** corruption: the reference parser scans lines in order and simply
  overwrites each field's value as it re-encounters that field name, so the
  **last** occurrence of a given field in the file wins. A reader must reproduce
  this exact behavior (last-occurrence-wins), not reject duplicates. (An earlier
  draft of this document flagged a cross-implementation divergence on this point —
  one implementation studied rejects duplicates instead. That implementation is
  the non-authoritative one for this spec; this rule now reflects the reference
  implementation's actual, verified behavior.)
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

- **TODO: verify** — Exact filenames (`.midge_leader`, `.midge_leader.lock`) as a
  required format detail versus convention; see §1.1.
- **TODO: verify** — Whether release forcing `acquired_at` to a sentinel-old
  timestamp (rather than deleting the record, or a dedicated "released" marker) is
  the intended shared contract; see §5.3.
- **TODO: verify** — Whether the missing checksum over the leader record is an
  accepted gap or an open format issue; see §7.
- **TODO: verify** — Whether any implementation requires an OS-level lock as
  defense-in-depth alongside the leader record; see §1.
