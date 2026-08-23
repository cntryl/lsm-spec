# Conformance

## Requirements

An implementation conforms to a given spec version when:

1. It correctly reads and writes every on-disk structure documented under
   `format/` at that version, per each document's normative rules — including its
   "format vs. policy" section, which distinguishes what's required from what's
   left to implementation choice.
2. It records the spec version (or repo tag/commit) it targets, in its own
   repository.

## Verification

- **Golden fixtures.** An implementation's writer output must be byte-identical to
  fixtures for a structure's deterministic fields (see each format doc for which
  fields are exempted as semantic-only), and its reader must correctly decode
  fixtures produced elsewhere.
- **Schema validation.** Where a `schema/*.json` definition exists for a structure,
  fixtures must validate against it.

## Status

No fixtures are published yet, so conformance cannot currently be verified against
this repository. The requirements above define what a fixture set must eventually
establish; adding one is tracked as outstanding work.

## Notes for fixture authors

- **Fixtures are writer output, not merely decodable input.** Where a document
  reserves a wire state — `Merge` entries and value-bearing `Delete` entries
  (`format/sst.md` §3.2, §3.4) — a reader must decode it, but no fixture representing
  conforming writer output may contain it. Reader-side fixtures exercising reserved
  or malformed states should be kept separate and labelled as such.
- **u64 bounds are not enforceable in JSON Schema.** The schemas declare
  `maximum: 18446744073709551615`, which is not representable as an IEEE 754 double.
  Validators that parse JSON numbers as doubles will compare against
  `18446744073709551616` instead, so a fixture one below `u64::MAX` may validate when
  it should not, and vice versa. Boundary cases at the top of the u64 range must be
  checked by the harness, not left to schema validation.
- **Cross-field equality is out of schema scope.** Several rules cannot be expressed
  in JSON Schema and must be checked by the harness: that a WAL catalog entry's map
  key equals its own `segment_id`, that its `object_key` equals the canonical key
  derived from its own `segment_id`/`writer_epoch` under the configured root
  (`format/wal.md` §1.2.1), and that every entry's `writer_epoch` is at most the
  catalog's `fencing_epoch`.
- **Binary formats have no schema.** The WAL frame and payload layouts, the SST
  block/entry/footer layouts, and the manifest journal's record framing are
  normative as byte layouts. The `schema/*.json` files covering them describe a
  *decoded* projection for cross-checking and are not a substitute for byte-level
  fixtures.
- **Beware `.gitignore`.** The repository ignores `*.log`, which would silently
  exclude a WAL fixture named `wal.log` — the format's own active-file name. Fixture
  directories must be exempted before any WAL fixtures are added.
