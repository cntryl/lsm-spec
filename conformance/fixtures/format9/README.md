# FORMAT 9 compatibility and schema examples

`empty-local/` and `populated-local/` are unchanged compatibility fixtures from
Midge main `c684310d9d8ce8ecd1151c47151d4bea98b348cf` (0.5.2). Source paths and
expected results are recorded in `fixtures.json`. The same bytes are present
in the 0.5.1 checkout; no re-encoding or FORMAT-marker substitution was used.

The `.json` control files are binary frames. The populated SST is V5; a strict
reader must return `fixture/alpha = value-alpha` across shutdown and reopen.
These are reader-compatibility cuts, not proof of canonical bytes for every
writer or coverage of cloud authority and nonempty journal replay.

`valid/` and `invalid/` hold decoded JSON schema examples authored for this
amendment. Their metadata fields use the manifest schema's hex-key projection.
The SST examples check logical version gates and deliberately use placeholder
handles; they are not full SST binaries and do not prove block tiling. State
transitions (live versus retired identity, stale moves, exact delete receipts)
remain reader/harness checks, beyond JSON Schema.
