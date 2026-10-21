# FORMAT 8 Manifest Fixtures

`populated-local/` is a populated FORMAT 8 compatibility fixture published in the
Midge `v0.4.1` source tree (`1bfa5c32e9300c4486c4969c386ca7dbc7622b24`). It contains
a FORMAT 8 marker, a manifest snapshot, a legacy `manifest.json` base, an empty
manifest journal, an empty intent log, and one SST. The reader must prefer the
snapshot when both manifest files exist. The legacy base is included to cover the
fallback shape; current FORMAT 8 writers do not maintain it. This fixture does not
cover WAL bytes or cloud catalog behavior.

`invalid/unsafe-sst-name.json` is schema-invalid because its SST name is not one safe
relative `.sst` filename. `invalid/missing-default-frontier.json` is a reader-only
malformed manifest: JSON Schema cannot enforce that the frontier list contains the
default family, so the manifest reader/harness must reject it.

The manifest JSON Schema checks field types and local constraints. The harness must
also check cross-field frontier ordering and family lifecycle rules from
`format/manifest.md` §5.7.
