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

### Open questions
Tracked as `TODO: verify` at the end of each `format/*.md` document. Not yet
resolved as of this version.
