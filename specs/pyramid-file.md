# The pyramid file

**Status:** version 1. Read and written by
`nonius-format` (`PyramidFile`, `write_pyramid`).

A pyramid file holds a series and the levels of its pyramid,
precomputed, so a server reads them in place instead of building them at
start. It is the format's, not the protocol's: no tile on the wire
changes, and a client never sees one.

It is made of `NSB1` containers ([the protocol](protocol.md), §2), each valid
on its own and starting at a multiple of 64 bytes, then a footer:

```
offset 0      the series container         the source file, byte for byte
  ↓ 64-aligned
              level container, first kept  one a kept level, finest first
              level container, next
              ...
  ↓ 64-aligned
              the index container          one row a level
              footer, 16 bytes             the file's last bytes
```

Gaps between the parts are zeros, written out; the file has no holes.

## The series

The series' own container, unchanged: its header, columns (`t`, `v`, `w`,
the Point profile) and metadata block. A reader of `NSB1` that knows
nothing of pyramids opens a pyramid file and reads the series — the
container's structure ends where its payload and metadata end, and what
follows is not its. A digest over the series' bytes is the source file's.

## A level

A container of eight columns, a column tile's ([the protocol](protocol.md), §4), by
the same names, types and exponents; one row a closed bucket of the
level, in time order:

| name | dtype | exponent |
|---|---|---|
| `first_t` | 1 (`i64`) | the series' `t` exponent |
| `first_v` | 1 | the series' `v` exponent |
| `last_t` | 1 | the series' `t` exponent |
| `last_v` | 1 | the series' `v` exponent |
| `min` | 1 | the series' `v` exponent |
| `max` | 1 | the series' `v` exponent |
| `count` | 2 (`u64`) | 0 |
| `weight` | 2 | 0 |

`flags` is 3. The metadata block is `{"pyramid":"level","level":L}`.

A level whose buckets hold about one point each summarises nothing and is
released: it has no container, and a reader answers it from
the series' points. Released levels are the finest ones; the top level is
always kept.

## The index

A container of eleven columns, one row a level, finest first — released
levels included:

| name | dtype | exponent | what |
|---|---|---|---|
| `width_ns` | 1 | the series' `t` exponent | the level's width; strictly increasing |
| `offset` | 2 | 0 | where its container starts; 0 when released |
| `len` | 2 | 0 | its container's length, to the end of its metadata; 0 when released |
| `open_first_t` … `open_weight` | as a level's | as a level's | the level's open bucket — the eight columns of a level, prefixed `open_`; `open_count` 0 when there is none |

The open buckets are the pyramid's tail: the points since each level's
last closed bucket. Released levels keep theirs, since a reader that goes
on pushing points needs every level's.

The metadata block is `{"pyramid":"index"}`.

## The footer

The file's last 16 bytes: the index container's offset, `u64`
little-endian, then the magic `NSBPYRA1`. A file that does not end in it
is not a pyramid file.

## What a reader checks

At every open, in time proportional to the number of levels — a bad file
is refused, never read out of bounds:

- the footer's magic, and the index's offset: 64-aligned, after the
  series, before the footer;
- the series, the index and every kept level's container, as any
  container is checked (the protocol, §2);
- the index's columns, and every level's, by name, type and exponent;
- widths strictly increasing; released levels first, the top kept;
  every kept level's container 64-aligned, after the one before it, and
  before the index.

On demand (`nonius-server verify`): every container's payload hash, and
the pyramid rebuilt from the points and compared bucket for bucket,
which also proves each level's buckets in order. Order is not checked at
open: it reads every level's `first_t`, the pages a file read in place is
built not to touch, and a level out of order gives wrong answers, not
unsafe ones.
