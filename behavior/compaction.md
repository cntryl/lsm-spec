# Compaction Behavior

Status: draft. Requirements use *must*, *must not*, *required*, *should*, and
*may* in the sense of RFC 2119.

## 1. Scope

This document defines data-authority rules for leveled compaction and source
fragmentation. It does not define planner scoring, work targets, output naming,
intent encoding, or a particular engine's error types.

## 2. Source fragmentation

Source fragmentation rewrites one SST into multiple SSTs at the same level to
reduce the work needed by a later merge. It changes file layout only; it is not
a merge or garbage-collection operation.

A fragment operation must preserve every source key/version entry, including
entries that are expired, deleted, or hidden by a newer version. It must also
preserve the coverage and sequence of every source range tombstone; a tombstone
may be split at output boundaries. The operation must not discard history based
on the current snapshot horizon.

Each output key range must remain within the source key range captured by the
plan. Outputs must preserve the planned interior cut: at least one output ends
at or before the cut and a following output starts at or after it. Output
ordering and range-boundary semantics must not lose or duplicate source point
entries.

## 3. Publication against current layout

Before a fragment replacement becomes authoritative, the writer must verify
that the captured source identity and the source/target topology still match
the plan, and that the completed outputs preserve the captured bounds and cut.
If any check fails, the writer must reject the replacement before publishing
it. The source and every unaffected SST must remain authoritative and readable;
the rejected outputs must not acquire publication authority.

Critical L0 work may arrive while a valid source fragment is being computed.
That work does not by itself invalidate the fragment. Once the fragment
publishes, the next compaction selection must prioritize draining the critical
L0 work before further inner-level promotion.

## 4. Recovery across manifest publication

The manifest replacement of a source by all of its fragment outputs is one
atomic publication. Recovery must use the authoritative manifest together with
the durable compaction intent to determine whether that replacement committed;
the intent phase alone is not sufficient.

If the manifest omits the source and contains every validated output, recovery
must retain the outputs and all source data represented by them, even if the
intent still records an earlier publication phase. If the manifest still
contains the source and omits every output in an `OutputDurable` intent, the
source remains authoritative and the unpublished outputs may be removed, as
specified in [cloud object-store behavior](cloud-storage.md#4-recovery-of-unpublished-compaction-outputs).
Partial or conflicting manifest visibility must follow the implementation's
ordinary recovery policy; recovery must not guess which copy contains the
committed data.

## 5. Implementation policy

The split-selection algorithm, output partition sizing, concurrency mechanism,
error type, and compaction intent representation are implementation policy.
Implementations may choose different policies while preserving the data,
publication, and recovery requirements above.

## Open questions

None for the behaviors specified here.
