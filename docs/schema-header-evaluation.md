# Schema Header Evaluation

This probe checks whether the compact front-header schema map is enough to
drive useful layout choices without hardcoded market-data schemas.

The probe used the actual Aura `records` writer/decoder plus the scoped grouped
writer in `src/scoped.rs`. Source-specific parsing stayed outside the repo in a
temporary harness.

## Header maps

```text
LTCUSDT 1m candles
  slots: ts, open, high, low, close, volume
  map:   255 0 2 2 2 0

Bybit ETHUSDT trades with UUID split into two i64 halves
  slots: ts, uuid_hi, uuid_lo, price, size, side, flag
  map:   255 0 0 0 0 0 0

Grimoire Bybit orderbook levels
  slots: ts, sequence_final, sequence_primary, side, price, qty_a, qty_b, flag
  map:   255 0 0 128 128 128 128 128
```

## Results

All rows round-tripped through decode checks with no corruption.

```text
dataset                         rows     raw i64     .aura       .aura0      .aura1      scoped grouped
LTCUSDT 90d 1m candles          129600   6220800     6221728     2349185     6220987     n/a
Bybit ETHUSDT tick sample       250000   14000000    14002784    7656424     10500179    n/a
Grimoire Bybit 900s book rows   79136    5064704     5067151     1751082     3640466     679447
```

Debug-build decode timings from the same run:

```text
dataset                         .aura0 decode   .aura1 decode   scoped grouped decode
LTCUSDT 90d 1m candles          191.1 ms        77.2 ms         n/a
Bybit ETHUSDT tick sample       497.7 ms        143.3 ms        n/a
Grimoire Bybit 900s book rows   141.7 ms        49.2 ms         77.9 ms
```

## Findings

For candles, the parent map is enough. The writer used the open/high/low/close
relationship to choose candle-shape field programs:

```text
ts      implicit fixed step
open    previous close offset
high    max(open, close) wick offset
low     min(open, close) wick offset
close   open-relative offset
volume  base-offset bitpack
```

For ticks, the plain root map is enough. The useful transforms are previous-row
or base-offset bitpacks, not same-row parent relationships. The UUID was split
into two i64 halves for this probe; stats must not attempt signed previous
deltas when adjacent opaque halves span more than the signed i64 delta range.
That overflow case is covered by a regression test.

For Grimoire-style orderbook rows, the `128` repeated-child scope is the missing
header signal. With only the old flat map, the columnar `.aura0` writer stores
duplicated event fields per level. With scoped repeated slots, the grouped writer
can write event fields once per websocket event and repeated fields once per
level. The grouped body was 679447 bytes versus 1751082 bytes for the current
columnar `.aura0` body+container path on the same decoded rows.

## Caveats

The Bybit tick run used the first 50000 trades from each public archive day
2026-06-09 through 2026-06-13. The full five daily archives are about 421 MB
compressed upstream, and the current generic i64 writer is still all-memory.
That is a writer scalability limitation, not a schema-header ambiguity.

The scoped grouped writer currently proves the header relationship is clear
enough and round-trips rows, but it is not yet the default `.aura0` compiled
profile path. To make it the profile path, the stamped footer needs a grouped
body instruction so the footer, not Rust-side dispatch, tells the decoder when
to use grouped event/child streams.
