# The Nonius protocol, version 1 — history in tiles

This document is the protocol. An implementation of either end — a
server that serves a series, or a client that draws one — is written from
it, not from Nonius's code. If the code and this document disagree, one of
them is wrong; the tests in `nonius-format` hold the worked examples below
to the bytes the code writes.

**Status:** version 1. The column tile's worked example, picking and
images (§9) were added within version 1: a client that does not ask for
images never sees them. It was clarified from the questions of a client
written from this document alone, without Nonius's code, which then
read, from a server it had never seen, exactly what Nonius draws.
`nonius-server` is the reference implementation of the server's side.

## 1. What it serves

A **series** is a sequence of points `(t, v, w)`, sorted by `t`:

| field | type | meaning |
|---|---|---|
| `t` | signed 64-bit integer | time, nanoseconds since the Unix epoch (UTC), non-decreasing |
| `v` | signed 64-bit integer | the value, scaled: the real value is `v · 10^-e`, `e` the series' value exponent |
| `w` | unsigned 64-bit integer | a weight: a volume, a sample count; 1 when the series has none |

A series can hold far more points than a client can. The server keeps a
**pyramid** of summaries of it at widths from fine to coarse, and serves
it in **tiles**: fixed, addressable pieces a client asks for when its view
needs them, like the tiles of a map. Version 1 serves a history that does
not change while it is served; a live tail is not part of it.

Conventions throughout: all binary integers are little-endian; time is
always nanoseconds since the epoch; division of times is **floor
division** (toward negative infinity), so times before 1970 work; in JSON,
every 64-bit integer is a **decimal string**, because JSON numbers lose
precision past 2^53.

## 2. The container, `NSB1`

Every binary payload is an `NSB1` container: a header, a directory of
columns, the columns' bytes, and an optional JSON metadata block.

```
offset 0      header, 64 bytes
offset 64     column directory, 48 bytes per column
              zero padding up to data_offset, a multiple of 64
data_offset   column 0's bytes, then zero padding to a multiple of 64
              column 1's bytes, then zero padding to a multiple of 64
              ...                                   <- payload_end
payload_end   metadata block, meta_len bytes of UTF-8 JSON (optional)
```

### Header — 64 bytes

| offset | size | field | value |
|---|---|---|---|
| 0 | 8 | `magic` | ASCII `NONIUSB1` |
| 8 | 2 | `version` | `1`; a reader rejects any other |
| 10 | 2 | `flags` | bit 0: the time column is non-decreasing; bit 1: it has no repeated value; other bits 0 |
| 12 | 2 | `column_count` | |
| 14 | 2 | — | 0 |
| 16 | 8 | `row_count` | every column has this many rows |
| 24 | 8 | `data_offset` | `align64(64 + 48 · column_count)` |
| 32 | 8 | `meta_offset` | where the metadata block starts, `payload_end`; 0 when there is none |
| 40 | 8 | `meta_len` | its length in bytes; 0 when there is none |
| 48 | 8 | `payload_hash` | FNV-1a-64 of the bytes `[data_offset, payload_end)`, padding included |
| 56 | 8 | — | 0 |

`align64(n)` rounds `n` up to a multiple of 64.

### Column directory entry — 48 bytes

| offset | size | field | value |
|---|---|---|---|
| 0 | 16 | `name` | ASCII, padded with zero bytes |
| 16 | 1 | `dtype` | 1 = signed 64-bit, 2 = unsigned 64-bit, 3 = 64-bit float |
| 17 | 1 | `exponent` | unsigned: the column's real value is `raw · 10^-exponent`; 9 for nanosecond times |
| 18 | 2 | — | 0 |
| 20 | 4 | — | 0 |
| 24 | 8 | `offset` | where the column's bytes start, a multiple of 64 |
| 32 | 8 | `len` | the column's length in bytes, `row_count · 8`, without padding |
| 40 | 8 | — | 0 |

The columns' bytes follow one another in directory order: column 0 at
`data_offset`, each next one at `align64` of the previous one's end.
`payload_end` is `align64` of the last column's end.

### FNV-1a-64

Start from `h = 0xcbf29ce484222325`; for each byte `b`, `h = (h XOR b) ·
0x100000001b3 mod 2^64`. Of the empty input it is `0xcbf29ce484222325`;
of the ASCII bytes `a` it is `0xaf63dc4c8601ec8c`.

### What a reader checks

A container comes from the network: a reader refuses one rather than
trusting it. It checks the magic and the version; that `data_offset` is
the value above; that every column's `offset` is where the layout rule
puts it and its `len` is `row_count · 8`; that the whole payload region
and the metadata block lie inside the bytes received, the block not
before `payload_end`; that every arithmetic step fits unsigned 64 bits;
and the payload hash. Each column is then read in place, as an array of
8-byte little-endian integers. Fields written as 0 may be ignored on
read. A tile is more than a container: a client also checks its columns
— names, types and exponents — against §4 or §5, and its metadata
against the address it asked for.

## 3. The pyramid

The pyramid has `levels` levels. Level `l` is made of **buckets** of
width `w_l = base · 4^l` nanoseconds, aligned to the epoch: bucket `j` of
level `l` covers `[j · w_l, (j + 1) · w_l)`. A bucket that holds no point
does not exist.

A bucket summarises the points in it as eight numbers:

| field | type | meaning |
|---|---|---|
| `first_t`, `first_v` | i64 | the first point's time and value |
| `last_t`, `last_v` | i64 | the last point's time and value |
| `min`, `max` | i64 | the least and greatest value |
| `count` | u64 | how many points, saturating at 2^64 − 1 |
| `weight` | u64 | the sum of their weights, saturating |

"First" and "last" are by position in the series, which is time order.
Two buckets `a` then `b`, every point of `b` after every point of `a`,
**merge** into one: `first_*` from `a`, `last_*` from `b`, `min` and `max`
the least and the greatest of the two, `count` and `weight` the
saturating sums.

### Choosing what a view reads

A client draws the time range `[start, end)` on `columns` pixel columns.
It wants columns `c = ceil((end − start) / columns)` nanoseconds wide,
and reads:

1. From the coarsest level `l` (counting down from `levels − 1`) with
   `k = floor(c / w_l) ≥ 16`: **columns** of `k` consecutive buckets of
   that level, merged. A column is `k · w_l` wide, aligned to the epoch:
   column `j` covers `[j · k · w_l, (j + 1) · k · w_l)`.
2. When not even level 0 gives `k ≥ 16`: the **raw points**, merged by
   the client into columns `c` wide aligned the same way.

The view's columns run from `floor(start / width)` to `floor((end − 1) /
width)`. Columns are whole and aligned, so the first and the last may
reach past `[start, end)`; and since `k` is rounded down they are at most
`c` wide — a view may have a few more columns than pixels (1,964 for a
day on 1,920 pixels, say). `k` is not bounded by the rule; a server
merges at most `max_k` (§6), so a view whose top level would need more —
far wider than the series — is not served, and a client keeps its views
within the series' span (Nonius allows a quarter more). To draw the line into and out of the view, a client also needs
the nearest non-empty column before the first and after the last: the
tiles carry them (§4, §5).

Drawn this way — a column's least to greatest value as a vertical span,
consecutive columns joined from one's last value to the next's first —
the line is the same, pixel for pixel, as the line drawn from every point
(the pyramid's M4 property).

## 4. Column tiles

A **column tile** is `N` consecutive columns of one level and one `k`:
tile `i` holds columns `[i · N, (i + 1) · N)`. `N` is the series'
`tile_columns`. Its address is `(series, level, k, i)`.

It is a container of eight columns, in this order:

| name | dtype | exponent |
|---|---|---|
| `first_t` | 1 | 9 |
| `first_v` | 1 | the value exponent |
| `last_t` | 1 | 9 |
| `last_v` | 1 | the value exponent |
| `min` | 1 | the value exponent |
| `max` | 1 | the value exponent |
| `count` | 2 | 0 |
| `weight` | 2 | 0 |

Each row is one non-empty column, merged from its `k` buckets, in time
order; empty columns have no row. A row's column index is
`floor(first_t / (k · w_l))`. Before the tile's own columns comes the
nearest non-empty column before the tile, if there is one; after them,
the nearest non-empty column after it, if there is one — told apart by
their index lying outside `[i · N, (i + 1) · N)`. So the nearest
non-empty column before a view is the last row, among the rows of the
view's first tile — its own and its neighbour — whose index is below
the view's first column; and after it, likewise in its last tile. `flags` is 3 (the rows'
`first_t` increases). The metadata block is
`{"tile":"columns","level":L,"k":K,"i":"I"}`, with `L` and `K` as JSON
numbers and `I` as a string, written exactly so — no spaces, keys in
that order; a client may compare it as text or as parsed JSON. A tile
always has it.

A tile outside the series' range is still a tile: it has no columns of
its own, and may carry neighbours.

### Worked example

A series of base width 1 ms, values with two decimal places, cut into
tiles of four columns (`N = 4`). Five points, at `t0 = 1788220800000000000`
(2026-09-01 00:00:00 UTC) and 3, 20, 21 and 70 ms after it, with values
108,000,000.00, .50, .20, 107,999,999.90 and 108,000,001.00, weights 3,
1, 2, 5, 4. At level 0 with `k = 16` a column is 16 ms wide; `t0` is in
column `t0 / 16000000 = 111763800000`, of tile `111763800000 / 4 =
27940950000`, which covers columns 111763800000 to 111763800003. The
first column merges the points at 0 and 3 ms, the second those at 20 and
21 ms; the point at 70 ms is in column 111763800004, past the tile — the
nearest non-empty column after it, its third row. Nothing comes before.

<!-- example: column-tile -->
```
0000: 4e 4f 4e 49 55 53 42 31 01 00 03 00 08 00 00 00
0010: 03 00 00 00 00 00 00 00 c0 01 00 00 00 00 00 00
0020: c0 03 00 00 00 00 00 00 35 00 00 00 00 00 00 00
0030: 66 01 95 ec 13 c4 90 e4 00 00 00 00 00 00 00 00
0040: 66 69 72 73 74 5f 74 00 00 00 00 00 00 00 00 00
0050: 01 09 00 00 00 00 00 00 c0 01 00 00 00 00 00 00
0060: 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0070: 66 69 72 73 74 5f 76 00 00 00 00 00 00 00 00 00
0080: 01 02 00 00 00 00 00 00 00 02 00 00 00 00 00 00
0090: 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00a0: 6c 61 73 74 5f 74 00 00 00 00 00 00 00 00 00 00
00b0: 01 09 00 00 00 00 00 00 40 02 00 00 00 00 00 00
00c0: 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00d0: 6c 61 73 74 5f 76 00 00 00 00 00 00 00 00 00 00
00e0: 01 02 00 00 00 00 00 00 80 02 00 00 00 00 00 00
00f0: 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0100: 6d 69 6e 00 00 00 00 00 00 00 00 00 00 00 00 00
0110: 01 02 00 00 00 00 00 00 c0 02 00 00 00 00 00 00
0120: 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0130: 6d 61 78 00 00 00 00 00 00 00 00 00 00 00 00 00
0140: 01 02 00 00 00 00 00 00 00 03 00 00 00 00 00 00
0150: 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0160: 63 6f 75 6e 74 00 00 00 00 00 00 00 00 00 00 00
0170: 02 00 00 00 00 00 00 00 40 03 00 00 00 00 00 00
0180: 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0190: 77 65 69 67 68 74 00 00 00 00 00 00 00 00 00 00
01a0: 02 00 00 00 00 00 00 00 80 03 00 00 00 00 00 00
01b0: 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
01c0: 00 00 f7 db 57 09 d1 18 00 2d 28 dd 57 09 d1 18
01d0: 80 1d 23 e0 57 09 d1 18 00 00 00 00 00 00 00 00
01e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
01f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0200: 00 ec ba 83 02 00 00 00 14 ec ba 83 02 00 00 00
0210: 64 ec ba 83 02 00 00 00 00 00 00 00 00 00 00 00
0220: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0230: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0240: c0 c6 24 dc 57 09 d1 18 40 6f 37 dd 57 09 d1 18
0250: 80 1d 23 e0 57 09 d1 18 00 00 00 00 00 00 00 00
0260: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0270: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0280: 32 ec ba 83 02 00 00 00 f6 eb ba 83 02 00 00 00
0290: 64 ec ba 83 02 00 00 00 00 00 00 00 00 00 00 00
02a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
02b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
02c0: 00 ec ba 83 02 00 00 00 f6 eb ba 83 02 00 00 00
02d0: 64 ec ba 83 02 00 00 00 00 00 00 00 00 00 00 00
02e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
02f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0300: 32 ec ba 83 02 00 00 00 14 ec ba 83 02 00 00 00
0310: 64 ec ba 83 02 00 00 00 00 00 00 00 00 00 00 00
0320: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0330: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0340: 02 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00
0350: 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0360: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0370: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0380: 04 00 00 00 00 00 00 00 07 00 00 00 00 00 00 00
0390: 04 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
03a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
03b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
03c0: 7b 22 74 69 6c 65 22 3a 22 63 6f 6c 75 6d 6e 73
03d0: 22 2c 22 6c 65 76 65 6c 22 3a 30 2c 22 6b 22 3a
03e0: 31 36 2c 22 69 22 3a 22 32 37 39 34 30 39 35 30
03f0: 30 30 30 22 7d
```

Read it as: the header — eight columns, three rows, `data_offset` 448
(`0x1c0`), the metadata at 960 (`0x3c0`) for 53 bytes; the directory of
eight entries, each column 24 bytes long; then `first_t`, `first_v`,
`last_t`, `last_v`, `min`, `max`, `count`, `weight`, three rows each,
each padded to 64 bytes; the metadata.

## 5. Raw tiles

A **raw tile** holds the points in `[i · R, (i + 1) · R)`, `R` the
series' `raw_tile_ns`. Its address is `(series, i)`; the tile holding
time `t` is `floor(t / R)`.

It is a container of three columns, `t` (dtype 1, exponent 9), `v`
(dtype 1, the value exponent) and `w` (dtype 2, exponent 0), one row a
point in series order. Before them comes the nearest point before the
range, after them the nearest point after it, if there are any — told
apart by their `t` lying outside the range. `flags` is 1 (`t`
non-decreasing), and also 2 when no two `t` are equal. The metadata block
is `{"tile":"raw","i":"I"}`.

### Drawing from raw points

A view drawn from raw points (§3, case 2) merges them into columns `c`
wide — whole columns, so the points of `[first · c, (last + 1) · c)` —
and its neighbour columns are merged **whole**, as the pyramid's
are: the nearest non-empty column before the view needs every point in
it, not only the one point nearest the view. So a client needs the raw
tiles that cover the view and also those that cover its two neighbour
columns. It finds them in two steps: the raw tiles of the view's span
give the nearest point before and after it (as neighbours, or as their
own points); the columns those points fall in are the neighbour
columns, and the raw tiles covering them are fetched too.

A raw tile's neighbours also prove tiles empty: if the nearest point
before tile `i` lies in tile `p`, every tile after `p` and before `i`
holds no point, and need not be fetched.

### Worked example

A series of base width 1 ms (`base = 1000000`) with `R = 2048 · 16 ·
base = 32768000000`, values with two decimal places. Two points:
`t = 1788220800000000000` (2026-09-01 00:00:00 UTC), `v = 10800000000`
(108,000,000.00), `w = 3`; and 1.5 µs later, `v = 10800000025`, `w = 1`.
Both lie in raw tile `floor(1788220800000000000 / 32768000000) =
54572167`; there are no points before or after them, so no neighbours.

<!-- example: raw-tile -->
```
0000: 4e 4f 4e 49 55 53 42 31 01 00 03 00 03 00 00 00
0010: 02 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00
0020: c0 01 00 00 00 00 00 00 1d 00 00 00 00 00 00 00
0030: 29 55 64 86 82 5a b8 b3 00 00 00 00 00 00 00 00
0040: 74 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0050: 01 09 00 00 00 00 00 00 00 01 00 00 00 00 00 00
0060: 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0070: 76 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0080: 01 02 00 00 00 00 00 00 40 01 00 00 00 00 00 00
0090: 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00a0: 77 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00b0: 02 00 00 00 00 00 00 00 80 01 00 00 00 00 00 00
00c0: 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0100: 00 00 f7 db 57 09 d1 18 dc 05 f7 db 57 09 d1 18
0110: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0120: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0130: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0140: 00 ec ba 83 02 00 00 00 19 ec ba 83 02 00 00 00
0150: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0160: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0170: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0180: 03 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00
0190: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
01a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
01b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
01c0: 7b 22 74 69 6c 65 22 3a 22 72 61 77 22 2c 22 69
01d0: 22 3a 22 35 34 35 37 32 31 36 37 22 7d
```

Read it as: the header (`0000`–`003f`) — magic, version 1, flags 3,
three columns, two rows, `data_offset` 256 (`0x100`), the metadata at 448
(`0x1c0`) for 29 bytes, the payload hash `0xb3b85a8286645529`; the
directory (`0040`–`00cf`) — `t` at 256, `v` at 320, `w` at 384, 16 bytes
each; zero padding to 256; the three columns, each padded to 64 bytes;
the metadata.

## 6. Series metadata

`GET /v1/series/{id}` answers `200` with `application/json`:

```json
{
  "protocol": 1,
  "series": "btcusdt",
  "base_ns": "1000000",
  "levels": 12,
  "ratio": 4,
  "tile_columns": 2048,
  "raw_tile_ns": "4096000000",
  "max_k": 4096,
  "v_exponent": 2,
  "rows": "1000000",
  "first_t": "1788220800000000000",
  "last_t": "1788307199999000000",
  "last": { "t": "1788307199999000000", "v": "10800001234", "w": "3" }
}
```

`ratio` is 4 in version 1. `first_t` and `last_t` are `null` for an
empty series, whose tiles are served all the same, empty. `last` is the
series' last point — its time, value (scaled as `v`) and weight, as
decimal strings — or `null` for an empty series: a client tags it on the
value axis, which it cannot do from tiles alone, since no tile says
where the series ends. Added within version 1: **a client ignores
fields it does not know**, and a server may leave out `last`, which a
client then reads as not known. The shape —
`base_ns`, `levels`, `tile_columns`, `raw_tile_ns`, `max_k` — is the
server's choice, and a client uses what the metadata says; the reference
server's default raw tile is `256 · 16 · base_ns`. `max_k` is the largest `k` the server merges; a server
chooses its `levels` so that the whole series, drawn on any screen, needs
no more at the top level.

## 7. HTTP

| request | answer |
|---|---|
| `GET /v1/series/{id}` | the metadata, `application/json` |
| `GET /v1/series/{id}/columns/{level}/{k}/{i}` | a column tile, `application/vnd.nonius.nsb1` |
| `GET /v1/series/{id}/raw/{i}` | a raw tile, `application/vnd.nonius.nsb1` |
| `GET /v1/series/{id}/pick?…` | the point under a pointer, `application/json` (§8) |
| `GET /v1/series/{id}/png?…` | the series drawn as an image, `image/png` (§9) |

`level`, `k` and `i` are decimal integers; `i` may be negative. A series
the server does not have is `404`. An address it cannot serve — a level
not below `levels`, a `k` below 1 or above `max_k`, a number that does
not parse — is `400`. A tile past the ends of the series is `200` with no
rows of its own.

Every tile has an `ETag`, the payload hash as 16 lowercase hex digits in
quotes; a request with a matching `If-None-Match` is answered `304`. In
version 1 a served history does not change, so every tile is sent with
`Cache-Control: public, max-age=31536000, immutable`. Answers carry
`Access-Control-Allow-Origin: *`, `Access-Control-Expose-Headers: ETag`
and `Cross-Origin-Resource-Policy: cross-origin`, so a page on another
origin — cross-origin isolated or not — can read them and their `ETag`;
and `Timing-Allow-Origin: *`, so that page's Resource Timing reports
what each answer cost.
The metadata is sent with `Cache-Control: no-cache`: a server may be
restarted on another history.

## 8. Picking

A client draws a crosshair and picks the point under the pointer. Near
the deepest zoom it holds the raw points of the pointer's columns and
picks among them; on a wide view those points are megabytes, so it asks
the server instead:

`GET /v1/series/{id}/pick?start=S&end=E&width=W&lo=L&hi=H&rows=R&column=C&row=Q&reach=K`

- `[start, end)` — the view, nanoseconds; `width` — the plot's columns.
- `lo`, `hi` — the value range on the plot, scaled as `v`; `rows` — the
  plot's rows. Value `v` falls on row `floor((v − lo)·(rows − 1) /
  (hi − lo))`, row 0 at the bottom.
- `column`, `row` — the pointer, from the plot's left and its bottom;
  `reach` — how many columns an empty column looks to either side.

All nine are decimal integers; a missing one is `400`. The answer is
`application/json`, `{"point":{"t":"T","v":"V","w":"W"}}` or
`{"point":null}`, and — the history being fixed — immutable, cached as
tiles are.

The point picked is Nonius's pick: among the points of the
pointer's column — its span of time, `[start + c·(end − start)/width,
start + (c + 1)·(end − start)/width)` — the one whose row is nearest the
pointer's, the latest of equals; an empty column reaches to the nearest
column holding a point within `reach`; with nothing within reach, a
pointer between two points picks the last one before it — the value in
force. The reference server answers with the same code that draws.

## 9. Images

A server may draw a series as an image — for a report, a thumbnail, a
message — with no browser:

`GET /v1/series/{id}/png?from=F&to=T&width=W&height=H[&theme=dark|light][&ratio=R][&gap=G][&tz=Z][&title=S][&unit=U]`

- `[from, to)` — the time shown, nanoseconds, `from` before `to`.
- `width`, `height` — the image, pixels, each 16 to 4096.
- `theme` — `dark` (the default) or `light`.
- `ratio` — device pixels a CSS pixel, above 0 and at most 8 (1 by
  default): labels and lines are drawn that many times larger.
- `gap` — nanoseconds, above 0: the line breaks wherever a point comes
  more than that after the one before. Joined by default.
- `tz` — an IANA time zone name, such as `Europe/Lisbon` (its `/` may
  come as `%2F`): the time labels in its local time, daylight saving
  included. UTC by default.
- `title`, `unit` — text in a header row above the plot: the title on
  the left, cut with "…" when too wide, the unit (such as `°C`) over the
  value labels. UTF-8, form-encoded as a browser's `URLSearchParams`
  writes it: a space as `+`, a plus as `%2B`, any other byte `%XX`. The
  labels draw Latin-1 and € − – — ‘ ’ “ ” … •; any other character as
  `?`. An empty text is none; the plot gives the row up while there is
  either, and `Nonius-Plot` says where it went. None by default.

The image is what a page of the Nonius package draws for a chart of this
one series on a canvas of that size, at that ratio, over that range,
with no pointer: its value range autoscaled over the range and padded,
the axes and labels drawn. The reference server draws it on the CPU with
the same code.

A missing or unreadable parameter, a side or a ratio out of range, a
`gap` of 0, a `tz` that names no zone, a `title` or `unit` that is not
percent-encoded UTF-8, an image too small at that ratio
to hold its axes and a plot, or `from` not before `to` is `400`, the
answer naming what was wrong; a range with no point in it is `404`. The
answer is `image/png`, 8-bit RGBA. Its `ETag` is the FNV-1a-64 of the
image's bytes, 16 lowercase hex digits in quotes: a matching
`If-None-Match` is answered `304`; and, the history being fixed, it is
cached as tiles are.

### Where the plot is

Every image, and every `304` for one, says where its plot is and what it
spans, in two headers readable from any origin:

- `Nonius-Plot: X Y PW PH` — the plot's rectangle in the image's
  pixels: its left column, its top row, its width and its height.
- `Nonius-Range: LO HI` — the values on the plot's bottom and top rows,
  scaled as `v`.

With `[from, to)`, these are exactly what a pick reads (§8): a pixel
`(px, py)` of the image is the pick's `column = px − X` and
`row = Y + PH − 1 − py`, with `start = from`, `end = to`, `width = PW`,
`lo = LO`, `hi = HI` and `rows = PH`. The other way, a point `(t, v)` is
on column `floor((t − from)·PW / (to − from))` and on row
`floor((v − LO)·(PH − 1) / (HI − LO))` from the bottom: something drawn
over the image at that pixel is where the line passes the point.

## 10. Versions

This is version 1: the `/v1/` in every path, `"protocol": 1` in the
metadata, `1` in every container's header. A change a version-1 client
cannot read is a new version.
