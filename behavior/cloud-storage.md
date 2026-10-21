# Cloud Object-Store Behavior

Status: draft. Requirements use *must*, *must not*, *required*, *should*, and
*may* in the sense of RFC 2119.

## 1. Scope

This document defines cloud object-store behavior needed to read immutable SST
objects consistently and recover interrupted compaction publication. Provider
request syntax may differ, but each provider must preserve the same observable
identity and recovery rules.

## 2. Identity-pinned range reads

Before reading ranges from an SST, a reader must obtain its total length and an
identity validator from the object store. Each range request must be conditional
on that observed identity. A provider-specific generation token or entity tag may
serve as the validator.

A successful range response must:

- have status `206 Partial Content`;
- identify the exact requested start and end offsets;
- report the same total object length observed before the first range; and
- contain exactly the requested number of bytes.

A provider response that omits or contradicts range metadata, ignores the range,
or returns a body of the wrong length is a protocol or I/O failure. It must not be
reported as an object-identity conflict.

If an object's identity changes before a conditional range completes, the reader
must stop and report a concurrent modification or precondition failure. It must
not combine bytes from different object identities. Once all ranges have
succeeded against the observed identity, a later replacement must not invalidate
the completed read.

## 3. GCS XML API bodyless requests

A request to the GCS XML API that has no payload must send `Content-Length: 0`.
This applies to bodyless `GET`, `HEAD`, and `DELETE` requests as well as other
bodyless operations. The request must not contain payload bytes.

## 4. Recovery of unpublished compaction outputs

A durable compaction intent in phase `OutputDurable` proves that its outputs were
not published when the authoritative manifest omits every output and still
references every input. In that state, the inputs remain the readable authority
and the output objects are disposable crash artifacts.

Recovery must remove the local and remote unpublished outputs even if an output's
local bytes are corrupt. Remote deletion must be conditional on the identity
observed for the object being removed. If that identity changes, recovery must
preserve the replacement and must not treat it as the old orphan.

This proven rollback must have the same data outcome under Strict and Salvage
recovery policies: retain every authoritative input, remove the unpublished
output copies, and keep all input data readable. If the intent, manifest, or
object identities do not prove that the output was unpublished, the reader must
use the ordinary recovery policy and must not infer that deletion is safe.

A retry may reuse an output name only after the prior object has been proven
removed. Publication must be conditional and must not overwrite an object whose
identity was not removed or verified.

## 5. WAL publication catalog recovery

The WAL publication catalog's primary object is authoritative. Its mirror is a
recovery copy, not a second source of truth. The object names and catalog data shape
are defined in [`format/wal.md`](../format/wal.md) §1.2.

When loading the catalog:

- If the primary exists and validates, use it and synchronize the mirror from the
  exact primary bytes. Do not select a newer-looking mirror over a valid primary.
- If the primary is missing, a valid mirror may restore it. If the primary is
  structurally corrupt, a valid mirror may also restore it. Restoring or synchronizing
  either copy must use an identity-conditional write against the object identity
  observed before the write.
- A primary that uses an unsupported catalog version is incompatible, not repairable
  corruption. If validation cannot complete because a resource limit was reached,
  that uncertainty also must not authorize replacement from the mirror. In either
  case, fail without changing either copy.
- If both copies are absent, no catalog exists. If neither copy validates, recovery
  fails; a reader must not guess which bytes represent the committed history.

For each catalog mutation, a writer must validate its current lease authority and
compare-and-swap the primary against the identity it read. Only after the primary
write is confirmed may it synchronize the mirror, using the confirmed primary bytes
and a conditional write against the mirror identity it observed. A failed or
ambiguous primary update must be resolved by rereading authority before retrying. If
the primary was confirmed but mirror synchronization failed, the primary remains
authoritative; the next successful load repairs the mirror from it.

An empty catalog may be created only as part of acquiring the authority that owns its
initial fencing epoch. Later mutations require an existing catalog and must not
silently initialize one.

## 6. Protocol requirements and implementation policy

The range status, offsets, total length, body length, observed object identity,
and GCS XML bodyless-request framing are protocol requirements. A conforming
implementation may choose its own internal error types, retry schedule, range
size, and storage layout, provided it preserves the observable rules above.

The compaction intent encoding and local cache layout are implementation policy.
They may differ between engines, provided recovery preserves the stated
authority, identity, and read-availability guarantees.

## Open questions

None for the behaviors specified here.
