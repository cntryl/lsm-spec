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
  fixtures must validate against it. A fixture index's `references` lists schemas
  required to resolve external `$ref` values (for example, the manifest edit schema
  reuses manifest definitions).

## Status

`fixtures/format9/` contains empty and populated FORMAT 9 compatibility bytes
copied unchanged from Midge 0.5.2 main at
`c684310d9d8ce8ecd1151c47151d4bea98b348cf`. The populated fixture covers one
SST V5, binary manifest bounds/frontiers, and intent suffix edits. The empty
fixture covers a kind-1 default manifest. `fixtures.json` records exact source
paths and expected reads; its JSON examples exercise new manifest/edit fields
and SST V5 version gates. They are decoded projections, not fabricated binary
writer output or a cloud qualification claim.

`fixtures/format8-manifest/populated-local/` remains historical FORMAT 8 data
from Midge v0.4.1. A FORMAT 9 reader must reject its marker without mutation.
Its JSON manifests can still be checked against the decoded projection schema;
that does not make their wire format compatible. Historical `format8-schemas/`
examples continue to exercise unchanged logical fields, WAL records, catalog
and ledger shapes.

There is not yet a full WAL segment/ledger byte fixture, cloud pointer/version
closure fixture, nonempty manifest-journal byte fixture, exhaustive intent
variant corpus, or cross-engine writer round-trip harness. Checksums, wire key
types, restart continuation correctness and chain invariants need binary reader
checks; schema success alone cannot establish them.

Malformed reader inputs are kept under `invalid/` and are not conformance writer
output. Invariants JSON Schema cannot express are explicitly left to an executable
conformance harness.

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
- **Cross-field and chain invariants are out of schema scope.** The harness must
  validate manifest frontier/family agreement, WAL ledger reachability and segment
  chain order, catalog anchor/segment continuity, monotonic epochs, retirement
  authority, and that `next_segment_id` exceeds every allocated id. A catalog
  segment object's key is derived from its epoch and segment id; no `object_key` is
  serialized in the catalog (`format/wal.md` §1.2.1).
- **Binary formats use decoded projections.** The WAL frame/payload layouts,
  SST block/entry/footer layouts, and the manifest journal's record framing are
  normative as byte layouts. The `schema/*.json` files covering them describe a
  *decoded* projection for cross-checking and are not a substitute for byte-level
  fixtures. FORMAT 9 manifest bounds project binary values to lowercase hex;
  allocator keys project unsigned integers to decimal strings. Cloud authority
  and intent variants do not yet have complete projection schemas.
- **Beware `.gitignore`.** The repository ignores `*.log`, which would silently
  exclude a historical fixture named `wal.log`. Current local WAL segments use
  final numeric names defined in `format/wal.md` §1.1. Exempt fixture directories
  before adding any `.log` fixtures.
