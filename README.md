# lsm-spec

Specification for the on-disk formats used by LSM-tree storage engines conforming to
this spec.

## Contents

- **`format/`** — on-disk byte layouts: WAL, SST, manifest, file lease.
- **`schema/`** — machine-checkable schema definitions mirroring `format/`.
- **`behavior/`** — behavioral contracts (conflict resolution, durability windows,
  recovery semantics). Not yet specified.
- **`conformance/`** — requirements for claiming and verifying conformance.

## Status

`0.1.0-dev` (see `VERSION`). Format documents are draft; open questions are marked
`TODO: verify` inline.

## Versioning

Semver. A breaking change to any normative rule is a major bump, an additive/
backward-compatible clarification is a minor bump, wording-only fixes are a patch
bump. See `CHANGELOG.md`.

## License

Licensed under the [Apache License, Version 2.0](LICENSE).
