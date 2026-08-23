# Sorted String Table (SST) Format

Status: draft, extracted from two independent implementations (a Rust engine and a C#
engine) for cross-checking. Sections marked "TODO: verify" reflect points where the
two implementations agree today but the spec author could not find an explicit,
independent design doc confirming the choice was deliberate (as opposed to
implementation-shared incidental behavior). This document assumes familiarity with
[`wal.md`](wal.md) and reuses shared primitives from
[`value-encoding.md`](value-encoding.md) rather than redefining them.

## 1. Overview

An SST file is an immutable, sorted, on-disk representation of a run of key/value
entries produced by flushing a memtable or by compaction. Once a writer finishes
producing an SST (`finalize`), the file never changes; readers only ever open it for
random point lookups, prefix/range scans, and periodic full verification.

At a high level, one SST file is a flat sequence of independently-framed **blocks**,
followed by a small number of **accelerator/metadata blocks**, followed by a
fixed-size **footer** at the very end of the file:

```
+-------------------+-------------------+-----+-------------------+
| data block 0      | data block 1      | ... | data block N-1    |
+-------------------+-------------------+-----+-------------------+
| range-tombstone block (optional)                                |
+-------------------------------------------------------------------+
| trie block (optional)                                            |
+-------------------------------------------------------------------+
| block-bloom block                                                 |
+-------------------------------------------------------------------+
| meta block (SstMetadata)                                          |
+-------------------------------------------------------------------+
| index block                                                       |
+-------------------------------------------------------------------+
| footer (fixed 84 bytes)                                            |
+-------------------------------------------------------------------+
```

A reader locates every block in the file by first reading the fixed-size footer at
EOF, which carries `BlockHandle`s (offset + size) for the meta block and the index
block, plus optional handles for the trie and block-bloom blocks. Everything else
(data block offsets, range-tombstone block location) is reached indirectly, through
the index block and the meta block respectively. There is no separate top-of-file
header or magic number — an SST is only self-identifying from the tail.

Entries within data blocks are sorted by key ascending, and for equal keys by
descending sequence number (newest version first). This ordering is what lets a
reader stop scanning a block as soon as it has seen the newest visible version of a
key.

### 1.1 What an SST does *not* store

- No embedded checksums per data block region beyond the whole-block compression
  trailer described in §2.3 — there is no separate per-entry checksum.
- No transaction/commit framing (that lives in the WAL). An SST entry only carries a
  single sequence number and operation type; multi-key atomicity is a WAL/memtable
  concept that has already been resolved by the time entries are flushed into an SST.

## 2. Terminology

| Term | Meaning |
|---|---|
| Block | A length-prefixed, individually compressed-and-checksummed unit of the file. Every block (data, range-tombstone, trie, block-bloom, meta, index) shares the same on-disk wrapping (§5.1). |
| Data block | A block holding a run of sorted, prefix-compressed key/value entries (§3). |
| Block handle | An `(offset, size)` pair identifying a block's position in the file. `size` includes the block's own length prefix and compression trailer (§5.1). |
| Index block | A block listing, for every data block, its first key and its block handle (§4.1). |
| Trie block | An optional accelerator block mapping structured key prefixes directly to a data-block ordinal, avoiding a full binary search of the index block (§4.2). |
| Block-bloom block | An optional block holding one Bloom filter per data block, for cheap negative point-lookups (§4.3). |
| Meta block | A block holding SST-level metadata: format version, chosen index kind, optional range-tombstone block handle, optional key-range bounds (§6). |
| Footer | The fixed-size trailer at EOF that a reader reads first, giving it the handles needed to locate the meta and index blocks (and, indirectly, everything else) (§7). |
| Restart point | A block-local point, once every `RESTART_INTERVAL` entries, where prefix compression resets by design so a reader can seek into the middle of a block without decoding entries from the start (see §3.3 — **TODO: verify** current wire use). |

## 3. Data block format

### 3.1 Block content: a sequence of entries

A decompressed data block's payload is simply a concatenation of variable-length
entries, back to back, with no block-level entry count or index — a reader consumes
entries sequentially until the decompressed byte range is exhausted. There is no
trailing restart-offset array inside the block itself (contrast with some other LSM
block formats, e.g. classic SSTable/RocksDB block footers) — **TODO: verify** whether
this is intentional (index-block-driven seeks make an in-block restart array
unnecessary because the accelerator structures in §4 always resolve to a specific
data block, then the reader linearly scans that one block) or a simplification that
both current engines happen to share.

### 3.2 Entry encoding

Each entry uses a packed binary layout (not TLV — TLV is reserved for block-level
metadata elsewhere in the file). Entry length is fully determined by its header, so
decoding is allocation-free and zero-copy.

Base header (26 bytes), little-endian:

| Field | Size | Offset | Description |
|---|---|---|---|
| `shared_prefix_len` | u16 | 0 | Number of leading bytes this entry's key shares with the *previous* entry's full key in this block (see §3.3). |
| `key_delta_len` | u16 | 2 | Length of the key suffix that follows (the non-shared remainder), unless equal to `0xFFFF` — see the extended form below. |
| `value_len` | u32 | 4 | Length of the value payload, unless `key_delta_len == 0xFFFF` and this field is `0xFFFFFFFF` — see the extended form below. |
| `sequence` | u64 | 8 | Sequence number for this entry (used for MVCC ordering and to pick the newest version of a key). |
| `entry_type` | u8 | 16 | One of `Put = 0`, `Insert = 1`, `Delete = 2`, `Merge = 3`. |
| `expiration_present` | u8 | 17 | `0` or `1`. Any other value is corruption. |
| `expiration_millis` | u64 | 18 | TTL expiration timestamp. Must be `0` when `expiration_present == 0` (non-canonical nonzero bytes with `expiration_present == 0` are treated as corruption, not silently ignored) — this makes "no TTL" and "TTL exactly at Unix epoch 0" and "TTL at `u64::MAX`" all distinguishable, unlike an earlier encoding that could not represent a `None`/`Some(u64::MAX)` distinction. |

If `key_delta_len == 0xFFFF` **and** `value_len == 0xFFFFFFFF` in the base header,
an **extended length block** immediately follows the base header, before the key
bytes:

| Field | Size | Description |
|---|---|---|
| `extended_key_delta_len` | u32 | Real key-delta length (used instead of the u16 field). |
| `extended_value_len` | u32 | Real value length (used instead of the u32 marker field). |

This lets a key delta exceed 65,535 bytes while keeping the common case's header at
26 bytes. The two marker values (`0xFFFF` / `0xFFFFFFFF`) are reserved sentinels, not
legitimate lengths — a real key-delta length of exactly 65,535 bytes must also use
the extended form (a writer promotes to the extended form at that boundary even
though 65,535 itself would technically fit the inline u16 field, to keep the marker
unambiguous).

Following the (base or extended) header:

```
[key_delta: key_delta_len bytes][value: value_len bytes, if present]
```

A `Delete` entry with `value_len == 0` carries no value bytes at all (the value is
absent, not merely empty) — see §3.4. All other entry types always carry a value
region of exactly `value_len` bytes, which may itself be zero-length (a legitimate
empty value, as opposed to Delete's absent value).

### 3.3 Key-delta (prefix) compression and restart points

Keys are stored as the delta relative to the **immediately preceding entry's full
key in the same block** (`shared_prefix_len` bytes are implicitly copied from that
previous key, then `key_delta` bytes are appended). The first entry of every data
block always encodes against an empty "previous key" (i.e. its `shared_prefix_len`
is 0 and its `key_delta` is its full key) — a data block is always independently
decodable from its own first byte without needing entries from a neighboring block.

`RESTART_INTERVAL = 16` is a defined constant governing "restart sampling of
prefix-compressed entries" on the write/read path. **TODO: verify**: the exact
reader-visible semantics of this constant were not confirmed from an independent
design doc — whether it currently does anything beyond bounding how many
sequentially-linked entries a scan must decode before re-verifying its position, or
whether it is fully vestigial today (since, per §3.1, there is no in-block restart
offset table that would let a reader jump to interval boundaries directly). Treat
the *value* 16 as informative but not yet a confirmed wire-format invariant.

### 3.4 Tombstones inside a data block

A `Delete`-type entry (`entry_type == 2`) represents a point tombstone for its key.
Whether it carries a zero-length value or an absent value is significant: a value is
present (even a zero-length one) for any entry type *other than* `Delete`, and for
`Delete` entries a value is present only if `value_len > 0` — in the normal case a
`Delete` is written with `value_len == 0` and therefore has *no* value region at all.

## 4. Accelerator and index structures

### 4.1 Index block (mandatory)

The index block is a flat, sorted list of `(first_key, block_handle)` pairs, one per
data block, in the same order as the data blocks appear in the file:

```
repeated {
  [key_len: u32 LE][key: key_len bytes][offset: u64 LE][size: u64 LE]
}
```

There is no entry count prefix; a reader consumes entries until the decompressed
index block's bytes are exhausted. Because entries are emitted in data-block order
and keys are globally sorted, the index block is itself sorted by `first_key` and
supports binary search for locating the containing data block of an arbitrary key
(fall back to the last block whose `first_key <= target`).

This structure is what the on-disk `IndexKind::Sparse` (value `0`) enum value names
— despite the historical name "Sparse," it is the **full** block-first-key binary
index, not a sampled/subsampled structure; a "sampled sparse-index accelerator" that
the name might suggest existed at some point but has been removed. **TODO: verify**
exact prior history — this document treats the current `Sparse == 0` behavior as the
only one that matters going forward.

### 4.2 Trie block (optional accelerator, real on-disk structure)

Unlike a purely in-memory optimization, the trie is a **persisted, on-disk block**
addressed by an optional `BlockHandle` in the footer (§7). Whether an SST carries a
trie block is chosen once, at write time, based on a key-structure profile (entropy,
average shared-prefix length, branching factor, etc. — write-time heuristic, out of
scope for wire format) and recorded in the meta block's `index_kind` field (§6) so a
reader knows whether to expect a trie handle. The chosen kind is a property of the
SST, not a per-read decision.

The trie is a compact radix trie over the same `first_key`s used to build the index
block; each leaf maps to a data-block *ordinal* (a 0-based index into the index
block's entry sequence, not a byte offset), so the reader still needs the index
block to translate a resolved ordinal into an actual `BlockHandle`. The trie is an
additional accelerator layered on top of the index block, not a replacement for it —
the index block is always present regardless of `index_kind`.

Encoding: a flat array of nodes (node 0 is always the root), varint-encoded:

```
[node_count: varint]
node* (node_count times):
  [prefix_len: varint]           -- length of prefix shared with parent, informational
  [key_delta_len: varint][key_delta: key_delta_len bytes]  -- this node's edge label
  [block_id: varint]             -- 0 = None (internal node), N = Some(N-1) (leaf, 0-based block ordinal)
  [child_count: varint]
  child* (child_count times):
    [first_byte: u8][child_index: varint]   -- index into the flat node array
```

Varints use the standard LEB128-style little-endian base-128 encoding (7 data bits
per byte, high bit set to continue). Children of a node are sorted ascending by
`first_byte` and a node may have at most one outgoing edge per byte value; readers
resolve a lookup by walking edges byte-by-byte and computing the longest-common-prefix
against each candidate child's key delta.

A conformant trie is validated as a proper tree before use: exactly one inbound edge
per non-root node, zero inbound edges to the root, no cycles, no unreachable nodes,
and every non-root node has a non-empty `key_delta`. A block-bloom or index-block
byte corruption is separable from a structurally-invalid trie graph — both are
treated as `Corruption`, but graph-shape validation happens independently of
checksum validation.

### 4.3 Block-bloom block (optional)

A block-bloom block holds one Bloom filter per data block (not one filter for the
whole SST), letting a reader skip an I/O for a candidate block once key-range and
index resolution have already narrowed the search to it, but before decompressing
and scanning that block:

```
[num_blocks: u32 LE]
[offset_0: u32 LE][offset_1: u32 LE]...[offset_{num_blocks-1}: u32 LE]
[bloom_data: concatenation of num_blocks serialized per-block Bloom filters]
```

`offset_i` is the byte offset of block *i*'s serialized filter within `bloom_data`
(not within the whole file). The per-block Bloom filter's own serialization format
(hash count, bit-array layout) is an implementation detail of the Bloom filter
sub-component — **TODO: verify** whether that inner format is itself meant to be
locked as part of this cross-engine spec or is considered a private accelerator
detail that a conformant reader is allowed to substitute (e.g. a reader without
Bloom support can simply skip this block entirely and fall back to reading every
candidate data block, since it is purely a negative-lookup accelerator with no
correctness impact if ignored).

## 5. Block wire format and compression

### 5.1 Persisted block wrapper

Every block in the file — data, range-tombstone, trie, block-bloom, meta, and index
— is written with the same length-prefixed wrapper:

```
+------------------+---------------------------------------------+
| payload_len (u32)| payload (payload_len bytes)                 |
| little-endian    | = compressed_block_data + trailer (§5.2)    |
+------------------+---------------------------------------------+
```

The `BlockHandle` recorded for this block (in the index block or the footer) uses:

- `offset`: the file offset of the 4-byte `payload_len` field itself (i.e. the start
  of the wrapper, not the start of the compressed payload).
- `size`: `4 + payload_len` — the *total* wrapper size, length prefix included.

A reader validates a handle by checking `offset + size <= block_region_end` (the
footer offset, i.e. everything before the footer itself) and by requiring
`size >= 4 + BLOCK_TRAILER_SIZE` (5 bytes, see §5.2) as a sanity floor before
attempting to decode.

### 5.2 Compression trailer

After compression, every block payload carries a 5-byte trailer:

```
[compressed_data][algo: u8][crc32c: u32 LE]
```

- `algo` is the compression-algorithm-ID byte defined once in
  [`value-encoding.md` §1](value-encoding.md#1-compression-algorithm-ids) — this
  document does not redefine that enum. An SST reader must reject an unknown `algo`
  byte as corrupt/incompatible rather than guessing.
- `crc32c` covers `compressed_data + algo` (i.e. everything in the trailer's block
  except the checksum field itself) — not the outer 4-byte length prefix from §5.1.

`BLOCK_TRAILER_SIZE = 5` (`1 + 4`) is a format constant.

### 5.3 Compression algorithm selection is policy, the trailer is format

*Whether* a given block ends up compressed, and by which of the algorithms in
`value-encoding.md`, is a writer-side policy decision (fixed algorithm, adaptive
try-several-and-keep-best, or never-compress) driven by a minimum input-size
threshold and a savings/ratio bar. None of that is part of the wire contract — only
the resulting `algo` byte and the trailer's presence/shape are. A reader must be
able to decompress a block regardless of which policy produced it. See §9.

## 6. Meta block

The meta block holds SST-level metadata not tied to any single data block. Layout,
little-endian:

| Field | Size | Description |
|---|---|---|
| `format_version` | u32 | Must equal the SST format version this document describes (**4**, "V4" — see §9 note on version numbering). Any other value is rejected: known-older versions are treated as an explicit incompatibility (`CompatibilityError`), not silently upgraded; there is no legacy decoder. |
| `index_kind` | u8 | `0` = `Sparse` (full binary index, §4.1 only), `1` = `Trie` (index block **and** trie block both present, §4.2). Any other value is corruption. |
| `flags` | u8 | Bit 0: key-range metadata present (§6.1). All other bits must be zero; a nonzero unknown bit is corruption, not a silently-ignored future extension. |
| *(reserved)* | 2 bytes | Must be zero. |
| `range_tombstone_handle` | 16 bytes (`offset` u64 + `size` u64) | `BlockHandle` for the range-tombstone block (§8), or all-zero if this SST carries no range tombstones. Half-present (nonzero size with zero offset, or zero size with nonzero offset) is corruption — the pair must be fully present or fully absent. |
| *(key-range, only if flag bit 0 set)* | variable | See §6.1. |

If the key-range flag is clear, the meta block must end exactly at the fixed 24-byte
header (any trailing bytes are corruption). If set, no trailing bytes are permitted
beyond the encoded key range either — the meta block's length is always fully
accounted for by its declared fields.

### 6.1 Key-range metadata (optional)

When present:

```
[smallest_key_len: u32 LE][smallest_key: smallest_key_len bytes]
[largest_key_len: u32 LE][largest_key: largest_key_len bytes]
```

`smallest_key` must be `<=` `largest_key` under byte-lexicographic ordering; an
inverted range is corruption. This range covers both real entries and any range
tombstones' `[start, end)` bounds (a writer folds tombstone bounds into the min/max
when computing the SST-level range) — it is a coarse gate for skipping whole SSTs
during a scan/compaction key-range check, not a substitute for the index block.

## 7. Footer

The footer is a fixed **84-byte** self-identifying trailer, always the last 84 bytes
of the file:

| Field | Size | Offset | Description |
|---|---|---|---|
| `meta_index_handle` | 16 bytes | 0 | `BlockHandle` (offset u64, size u64) for the meta block (§6). |
| `index_handle` | 16 bytes | 16 | `BlockHandle` for the index block (§4.1). |
| `trie_handle` | 16 bytes | 32 | `BlockHandle` for the trie block (§4.2), or all-zero if absent. |
| `block_bloom_handle` | 16 bytes | 48 | `BlockHandle` for the block-bloom block (§4.3), or all-zero if absent. |
| `format_version` | u32 | 64 | Must equal **4**. |
| `footer_size` | u32 | 68 | Must equal **84** (the footer's own encoded size — self-describing, so a future footer revision could in principle change size without breaking a reader that trusts this field over a hardcoded constant, though the current reader in fact hardcodes 84 and treats any other declared value as corruption rather than as a hint to re-read a differently-sized region). |
| `magic` | u64 | 72 | Fixed magic number `0xDB47_7524_8B80_FB57` (chosen to be RocksDB-SST-footer-compatible in value, not merely coincidentally similar — **TODO: verify** whether byte-for-byte RocksDB footer compatibility beyond this one magic value is an actual design goal or purely incidental reuse of a well-known constant). |
| `crc32c` | u32 | 80 | CRC32C over footer bytes `[0, 80)` — i.e. every preceding footer field, not including this checksum itself. |

Total footer size: **84 bytes**, always, regardless of which optional handles are
populated (unused handles are zero-filled, not omitted — the footer never changes
length).

### 7.1 Locating the index from EOF

A reader opens an SST by:

1. Reading the file's total length.
2. Reading the last 84 bytes as the footer.
3. Validating `magic` and `crc32c` first. A magic mismatch or checksum failure at
   this stage is `Corruption`. **Exception**: if the file is *shorter* than 84 bytes,
   or the 84-byte footer fails to decode, a reader additionally checks whether the
   last 8 bytes of the file equal the magic value alone — if so, this indicates a
   pre-V4 (V1–V3) footer shape from an older, incompatible format generation, and
   must be reported as `CompatibilityError` ("unsupported old format"), distinct
   from ordinary corruption.
4. Validating `format_version == 4` (`CompatibilityError` if not — including
   both older and unknown-future versions; there is no forward-compatible
   best-effort read path).
5. Using `meta_index_handle` and `index_handle` (and, if present, `trie_handle` /
   `block_bloom_handle`) to read those blocks by direct offset — no further
   sequential scanning of the file is required. `block_region_end` for handle
   bounds-checking purposes is the footer's own starting offset
   (`file_length - 84`); every block handle in the file must resolve to a byte
   range strictly within `[0, block_region_end)`.
6. Reading the meta block to learn `index_kind`, whether a range-tombstone block
   is present, and the key range.
7. Reading the index block to build the sorted `(first_key, handle)` table used for
   data-block resolution during point lookups and scans.

No block in the file needs to be read in file order; every block is reached by a
direct handle, either from the footer or (for data blocks) from the index block.

## 8. Range tombstones

An SST may carry range tombstones — deletions of a `[start, end)` key range rather
than a single key — in a dedicated, optional block whose handle is recorded in the
meta block (§6), not the footer. Encoding:

```
[count: u32 LE]
repeated count times:
  [start_len: u32 LE][start: start_len bytes]
  [end_len: u32 LE][end: end_len bytes]
  [seq: u64 LE]
```

- The range is half-open: `start` is inclusive, `end` is exclusive. A tombstone
  covers key `k` iff `start <= k < end` (so `start == end` covers nothing).
- `seq` is the sequence number the tombstone was written at, used the same way a
  point tombstone's sequence is: only a read whose visible sequence range extends
  past `seq` observes the deletion.
- Range tombstones are stored separately from data-block entries — they are not
  interleaved with point entries in data blocks, and are not indexed by the trie or
  block-bloom accelerators (those only cover point-entry keys). A reader must
  additionally consult this block (when present) for every read that could fall
  inside a covered range; this is a correctness-relevant scan, not an optional
  accelerator.

## 9. Format vs. policy — what's in scope here

The following are explicitly **behavioral/policy**, not part of the on-disk format
contract, and are not required for wire compatibility between two engines that both
correctly implement §1–§8:

- **Target/maximum block size** (the size threshold at which a writer closes the
  current data block and starts a new one): a writer-side tuning knob. A reader
  never needs to know the target size a writer used — it only needs the block
  boundaries the writer actually chose, which are recorded in the index block.
- **Compression policy selection** (fixed algorithm vs. adaptive try-several,
  minimum-input-size threshold, savings/ratio bar for adaptive selection): writer
  heuristic. In scope: the resulting `algo` byte and trailer shape (§5.2), which
  every reader must be able to decode regardless of which policy chose it.
- **`IndexKind` auto-tuning heuristic** (the entropy/branching-factor/key-length
  thresholds that decide `Sparse` vs. `Trie` for a given SST): writer-side policy.
  In scope: that the chosen kind is recorded in the meta block and that both index
  representations (§4.1, §4.2) must be correctly readable — a reader must handle
  either kind, but never needs to reproduce the *decision* itself.
- **Restart interval value tuning** (§3.3): to the extent `RESTART_INTERVAL` governs
  only a write/scan performance characteristic and not a decodable wire structure,
  its numeric value is policy, not format. Flagged **TODO: verify** above because it
  isn't fully confirmed that it has zero reader-visible effect on wire bytes.
- **Bloom filter false-positive rate / hash-function choice / bits-per-key** for the
  per-block filters in §4.3: writer-side accelerator tuning. A reader that cannot or
  chooses not to interpret the inner Bloom filter format can skip the block-bloom
  block entirely with no correctness impact, only a possible performance cost from
  reading blocks that a filter would have let it skip.
- **When compaction chooses to rewrite/merge SSTs**, and how many/which SSTs a given
  compaction picks: entirely a policy/scheduling concern one layer above the file
  format described here.

Ambiguous / needs a decision when writing the canonical spec:

- **TODO: verify** — Whether byte-for-byte compatibility with RocksDB's SST footer
  beyond the shared magic constant (§7) is an actual design goal, or the magic value
  was borrowed opportunistically with no further compatibility intended. This
  matters for whether future footer fields may freely diverge from RocksDB's own
  footer layout.
- **TODO: verify** — Whether `RESTART_INTERVAL = 16` (§3.3) has any reader-visible
  wire effect today, given data blocks appear to have no in-block restart-offset
  table (§3.1). If it truly has none, it should be documented purely as a writer/
  scan-loop implementation detail with no format-compatibility weight, not as part
  of this spec's normative sections.
- **TODO: verify** — Whether the per-block Bloom filter's inner serialization
  (§4.3) is meant to be locked as a cross-engine wire format, or is intentionally
  left as a swappable accelerator detail (since ignoring it entirely remains
  correct, only slower).
- **TODO: verify** — Whether a future format revision is expected to bump
  `format_version` past 4 with a defined migration/coexistence story, or whether
  (as with the V1–V3 -> V4 transition) the expectation is a hard break with export/
  import as the only supported upgrade path. `docs/development/format-compatibility.md`
  in the Rust engine's repo documents the V4 transition as exactly that kind of hard
  break; whether that precedent is meant to generalize to all future bumps is not
  independently confirmed.
