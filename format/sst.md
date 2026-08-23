# Sorted String Table (SST) Format

Status: draft. Sections marked "TODO: verify" record points that are not yet settled;
they are open questions for this specification, not statements about any particular
implementation. Requirements are expressed with must / must not / required / should /
may in the sense of RFC 2119, whether or not they appear in capitals.

This document assumes familiarity with [`wal.md`](wal.md) and reuses shared
primitives from [`value-encoding.md`](value-encoding.md) rather than redefining them.

## 1. Overview

An SST file is an immutable, sorted, on-disk representation of a run of key/value
entries produced by flushing a memtable or by compaction. Once a writer finishes
producing an SST (`finalize`), the file never changes; readers only ever open it for
random point lookups, prefix/range scans, and periodic full verification.

At a high level, one SST file is a flat sequence of independently-framed **blocks**,
followed by a small number of **accelerator/metadata blocks**, followed by a
fixed-size **footer** at the very end of the file:

```
+------------------+------------------+-----+------------------+
| data block 0     | data block 1     | ... | data block N-1   |
+------------------+------------------+-----+------------------+
| range-tombstone block (optional)                             |
+--------------------------------------------------------------+
| trie block (optional)                                        |
+--------------------------------------------------------------+
| block-bloom block (optional)                                 |
+--------------------------------------------------------------+
| meta block                                                   |
+--------------------------------------------------------------+
| index block                                                  |
+--------------------------------------------------------------+
| footer (fixed 84 bytes)                                      |
+--------------------------------------------------------------+
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

- No checksums within a data block beyond the whole-block compression trailer
  described in §5.2 — there is no per-entry checksum.
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

## 3. Data block format

### 3.1 Block content: a sequence of entries

A decompressed data block's payload is simply a concatenation of variable-length
entries, back to back, with no block-level entry count or index — a reader consumes
entries sequentially until the decompressed byte range is exhausted. There is no
trailing restart-offset array inside the block itself (contrast with some other LSM
block formats, e.g. classic SSTable/RocksDB block footers) — **TODO: verify** whether
this is intentional (index-block-driven seeks make an in-block restart array
unnecessary because the accelerator structures in §4 always resolve to a specific
data block, then the reader linearly scans that one block) or an omission worth
revisiting.

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

**`Merge` (`entry_type = 3`) is RESERVED as of this spec version.** This document
defines its wire encoding (identical framing to `Put`/`Insert`, so a reader must
still be able to parse one structurally) but no merge-operator semantics exist yet
anywhere in this specification — how multiple `Merge` operands for the same key would
combine into a resolved value, and whether that resolution happens at read time or
at compaction time, are both undefined. No operation in [`wal.md`](wal.md)'s `OP`
registry (§5.1) produces a `Merge` record either, so there is no defined path from a
conforming WAL writer to a `Merge` SST entry in the first place. **A conforming
writer MUST NOT emit `entry_type = 3` until a future revision of this spec defines
merge-operator semantics.**

A reader MUST treat the extended form as in use if and only if **both**
`key_delta_len == 0xFFFF` **and** `value_len == 0xFFFFFFFF` hold in the base header
— this conjunctive (both-fields) check, not a single-field check against
`key_delta_len` alone, is the reader-side trigger. When it holds, an **extended
length block** immediately follows the base header, before the key bytes:

| Field | Size | Description |
|---|---|---|
| `extended_key_delta_len` | u32 | Real key-delta length (used instead of the u16 field). |
| `extended_value_len` | u32 | Real value length (used instead of the u32 marker field). |

This lets a key delta exceed its inline u16 field's range and lets the marker pair
represent legitimate lengths while keeping the common case's header at 26 bytes.
Because both extended lengths are u32 values, neither real length may exceed
4,294,967,295 bytes (`u32::MAX`); a writer MUST reject an entry whose key delta or
value exceeds that limit.

**Writer requirement — precise version** (revised; see the note below on why an
earlier version of this rule was stricter than necessary): a writer MUST use the
extended form whenever any of the following hold, and MAY use it more liberally at
its own discretion:
- the real key-delta length exceeds 65,535 bytes (physically doesn't fit the
  inline u16 field), or
- the real key-delta length is *exactly* 65,535 bytes **and** the real value
  length is *exactly* 4,294,967,295 bytes, simultaneously — the one combination
  that would otherwise be indistinguishable from the extended-form marker pair
  itself if written inline.

A writer is **not** required to promote to the extended form solely because the
real key-delta length happens to equal exactly 65,535 bytes, as long as the real
value length is not *also* exactly 4,294,967,295 at the same time — writing
`key_delta_len = 0xFFFF` as a literal, ordinary length is safe and correctly
decodable under the conjunctive reader rule above, precisely because the reader
only treats it as "extended in use" when `value_len` is *also* the sentinel.

An earlier revision of this document stated a stricter, single-field rule; see
`CHANGELOG.md`.

Following the (base or extended) header:

```
[key_delta: key_delta_len bytes][value: value_len bytes, if present]
```

A `Delete` entry with `value_len == 0` carries no value bytes at all (the value is
absent, not merely empty) — see §3.4. All other entry types always carry a value
region of exactly `value_len` bytes, which may itself be zero-length (a legitimate
empty value, as opposed to Delete's absent value).

### 3.3 Key-delta (prefix) compression

Keys are stored as the delta relative to the **immediately preceding entry's full
key in the same block** (`shared_prefix_len` bytes are implicitly copied from that
previous key, then `key_delta` bytes are appended). The first entry of every data
block always encodes against an empty "previous key" (i.e. its `shared_prefix_len`
is 0 and its `key_delta` is its full key) — a data block is always independently
decodable from its own first byte without needing entries from a neighboring block.

This format defines **no restart points**. There is no in-block restart offset table
(§3.1) and no reader-visible wire structure that a restart interval could govern. A
writer that maintains one internally, for its own scan loop, produces no
distinguishing bytes; see §9.

### 3.4 Tombstones inside a data block

A `Delete`-type entry (`entry_type == 2`) represents a point tombstone for its key.
Whether it carries a zero-length value or an absent value is significant: a value is
present (even a zero-length one) for any entry type *other than* `Delete`, and for
`Delete` entries a value is present only if `value_len > 0` — in the normal case a
`Delete` is written with `value_len == 0` and therefore has *no* value region at all.

**A `Delete` entry with `value_len > 0` is RESERVED as of this spec version.**
[`wal.md`](wal.md) §5.1 defines the WAL's `Delete` operation as always value-less
(`VALUE` TLV tag never permitted), and the WAL, via memtable flush, is the only
mutation source this specification defines flowing into an SST — so there is no
defined path from a conforming WAL writer to a value-bearing `Delete` SST entry.
**A conforming writer MUST NOT set `value_len > 0` on a `Delete` entry** until a
future revision of this spec defines a use for it; a reader must still decode the
wire state correctly (per §3.2's general rule) for forward compatibility with a
future revision that does define one.

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
and a non-empty `key_delta` on every non-root node. Graph-shape validation is
independent of checksum validation — a block whose trailer checksum verifies may
still decode to a structurally invalid graph. Both failures are reported as
`Corruption`.

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
locked as part of this cross-implementation contract or is considered a private accelerator
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

A reader validates a handle by checking `offset + size <= block_region_end` — where
`block_region_end` is the footer's own starting offset, i.e. the end of everything
that precedes the footer — and by requiring `size >= 4 + BLOCK_TRAILER_SIZE` (that
is, at least 9 bytes: the 4-byte length prefix plus the 5-byte trailer of §5.2) as a
sanity floor before attempting to decode.

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
2. Reading the last 84 bytes as the footer, if the file is at least 84 bytes long.
3. Locating and validating the footer, in this order (the legacy check in (b) always
   runs before a magic/checksum failure is reported as ordinary corruption — it is
   not a competing, independent rule):
   a. If the file is at least 84 bytes long, validate the footer's `magic` and
      `crc32c` (§7). If both validate, proceed to step 4.
   b. If the file is *shorter* than 84 bytes, or step (a)'s validation failed (magic
      mismatch or checksum failure), check whether the file is at least 8 bytes long
      and whether its *last 8 bytes* equal the magic value alone
      (`0xDB47_7524_8B80_FB57`, §7). If so, this indicates a pre-V4 (V1–V3) footer
      shape from an older, incompatible format generation, and must be reported as
      `CompatibilityError` ("unsupported old format"), distinct from ordinary
      corruption.
   c. If neither (a) nor (b) succeeds — the file is too short even for the 8-byte
      legacy check, or the last 8 bytes don't equal the magic value either — the
      failure is `Corruption`.

   A reader must not treat the bare-magic check as an independent rule competing
   with the full footer decode: it runs only as a fallback, only after (a) has been
   attempted and failed or was impossible.
4. Validating `format_version == 4` (`CompatibilityError` if not — including
   both older and unknown-future versions; there is no forward-compatible
   best-effort read path).
5. Using `meta_index_handle` and `index_handle` (and, if present, `trie_handle` /
   `block_bloom_handle`) to read those blocks by direct offset — no further
   sequential scanning of the file is required. `block_region_end` for handle
   bounds-checking purposes is the footer's own starting offset
   (`file_length - 84`); every block handle in the file must satisfy §5.1's
   `offset + size <= block_region_end`.
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
- **Restart interval** (§3.3): a writer's internal restart interval, if it keeps one,
  governs no decodable wire structure and carries no format-compatibility weight.
- **Bloom filter false-positive rate / hash-function choice / bits-per-key** for the
  per-block filters in §4.3: writer-side accelerator tuning. A reader that cannot or
  chooses not to interpret the inner Bloom filter format can skip the block-bloom
  block entirely with no correctness impact, only a possible performance cost from
  reading blocks that a filter would have let it skip.
- **When compaction chooses to rewrite/merge SSTs**, and how many/which SSTs a given
  compaction picks: entirely a policy/scheduling concern one layer above the file
  format described here.

**Settled:**

- `Merge` (`entry_type = 3`, §3.2) is reserved as of this spec version: no
  merge-operator contract exists anywhere in this specification, and no WAL
  operation produces one. The reservation should be lifted in favor of a real
  specification, rather than standing indefinitely, once merge semantics are
  defined.
- A `Delete` entry with `value_len > 0` (§3.4) is reserved as of this spec version:
  no defined use case exists for it, and no WAL operation produces one.
- The extended-length promotion boundary (§3.2): the reader's conjunctive check
  governs, and the minimal writer requirement follows from it. An earlier revision
  stated a stricter single-field rule, now withdrawn; see `CHANGELOG.md`.

**Open questions:**

- **TODO: verify** — Whether byte-for-byte compatibility with RocksDB's SST footer
  beyond the shared magic constant (§7) is an actual design goal, or the magic value
  was borrowed opportunistically with no further compatibility intended. This
  matters for whether future footer fields may freely diverge from RocksDB's own
  footer layout.
- **TODO: verify** — Whether the per-block Bloom filter's inner serialization
  (§4.3) is meant to be locked as a cross-implementation wire format, or is
  intentionally left as a swappable accelerator detail (since ignoring it entirely
  remains correct, only slower).
- **TODO: verify** — Whether a future revision is expected to bump `format_version`
  past 4 with a defined migration/coexistence story, or whether the expectation is a
  hard break with export/import as the only supported upgrade path, as the V1–V3 to
  V4 transition was. Whether that precedent generalizes to all future bumps is
  unsettled.
