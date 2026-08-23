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
