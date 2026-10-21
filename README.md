# lsm-spec

Specification for the on-disk formats used by LSM-tree storage engines conforming to
this spec.

This repository is the authority for these formats. It defines them in their own
terms, independently of any implementation.

## Authority and provenance

The specification is being reconstructed from existing implementations, but that is
research provenance, not an authority relationship. Midge, Pants, and other engines
provide evidence about historical intent and interoperability; none defines the
format. Normative requirements flow from this specification to implementations.

When an implementation differs from a sound rule, the implementation must change.
When investigation shows a rule itself is wrong, the specification must be amended
deliberately, with rationale and compatibility impact recorded. Normative text must
not be silently changed merely to match current code. Implementation-specific
findings belong in `notes/`, which is non-normative.

## Contents

- **`format/`** — on-disk byte layouts: WAL, SST, manifest, file lease, and the
  shared value-encoding primitives they share.
- **`schema/`** — machine-checkable schema definitions mirroring `format/`.
- **`behavior/`** — behavioral contracts (conflict resolution, durability windows,
  recovery semantics). Cloud object-store identity and publication recovery are
  specified in `behavior/cloud-storage.md`.
- **`conformance/`** — requirements for claiming and verifying conformance.
- **`notes/`** — non-normative working notes. Not part of the specification.

## Conventions

The words *must*, *must not*, *required*, *should*, and *may* are used in the sense
of RFC 2119 wherever they state a requirement, whether or not they appear in
capitals.

Each format document ends with a **format vs. policy** section separating what an
implementation must reproduce for wire compatibility from what it is free to decide.
Points that remain unsettled are marked `TODO: verify` inline and collected under
that section's **Open questions**.

## Status

`1.0.0-dev` (see `VERSION`). Format documents define the draft FORMAT 8 profile:
manifest FORMAT 8, WAL payload/ledger/segment versions 2/2/2, SST V4, and cloud WAL
catalog v3. The conformance set includes a populated FORMAT 8 compatibility fixture
and schema-validation examples; it is not yet a complete byte-level conformance suite.
See `conformance/README.md`.

## Versioning

Semver. A breaking change to any normative rule is a major bump, an additive/
backward-compatible clarification is a minor bump, wording-only fixes are a patch
bump. See `CHANGELOG.md`.

## License

Licensed under the [Apache License, Version 2.0](LICENSE).
