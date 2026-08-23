# lsm-spec

Specification for the on-disk formats used by LSM-tree storage engines conforming to
this spec.

This repository is the authority for these formats. It defines them in their own
terms, independently of any implementation.

## Contents

- **`format/`** — on-disk byte layouts: WAL, SST, manifest, file lease, and the
  shared value-encoding primitives they share.
- **`schema/`** — machine-checkable schema definitions mirroring `format/`.
- **`behavior/`** — behavioral contracts (conflict resolution, durability windows,
  recovery semantics). Not yet specified.
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

`0.1.0-dev` (see `VERSION`). Format documents are draft. No conformance fixtures are
published yet; see `conformance/README.md`.

## Versioning

Semver. A breaking change to any normative rule is a major bump, an additive/
backward-compatible clarification is a minor bump, wording-only fixes are a patch
bump. See `CHANGELOG.md`.

## License

Licensed under the [Apache License, Version 2.0](LICENSE).
