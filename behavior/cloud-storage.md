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

## 5. Protocol requirements and implementation policy

The range status, offsets, total length, body length, observed object identity,
and GCS XML bodyless-request framing are protocol requirements. A conforming
implementation may choose its own internal error types, retry schedule, range
size, and storage layout, provided it preserves the observable rules above.

The compaction intent encoding and local cache layout are implementation policy.
They may differ between engines, provided recovery preserves the stated
authority, identity, and read-availability guarantees.

## Open questions

None for the behaviors specified here.
