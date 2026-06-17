# Schema Header Evaluation

This probe checks whether the compact front-header schema map is enough to
drive useful layout choices without hardcoded market-data schemas.

The probe used the actual Aura `records` writer/decoder plus the scoped grouped
writer in `src/scoped.rs`. Source-specific parsing stayed outside the repo in a
temporary harness.

## Legacy probe maps

These measurements used the earlier compact map where `255` marked timestamp
and `128..254` marked repeated child scope. The results are still useful as
transform evidence, but the byte assignments are not the current v1 schema
dialect. The current dialect is documented in `docs/schemas.md`.

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

Grimoire Bybit orderbook levels, parented repeated slots
  slots: ts, sequence_final, sequence_primary, side, price, qty_a, qty_b, flag
  map:   255 0 0 128 132 133 133 133
```

Current-byte equivalents:

```text
LTCUSDT 1m candles:               100 0 2 2 2 0
Bybit ETHUSDT trades:             100 0 0 0 0 0 0
Grimoire orderbook levels:        100 0 0 205 0 0 0 0
Parented Grimoire orderbook rows: 100 0 0 205 4 5 5 5
```

## Results

All rows round-tripped through decode checks with no corruption.

```text
dataset                         rows     raw i64     .aura       .aura0      .aura1      scoped grouped
LTCUSDT 90d 1m candles          129600   6220800     6221728     2349185     6220987     n/a
Bybit ETHUSDT tick sample       250000   14000000    14002784    7656424     10500179    n/a
Grimoire Bybit 900s book rows   79136    5064704     5067151     1751082     3640466     679447
Grimoire parented grouped run   79136    5064704     5067667     276998      3640834     n/a
```

Debug-build decode timings from the same run:

```text
dataset                         .aura0 decode   .aura1 decode   scoped grouped decode
LTCUSDT 90d 1m candles          191.1 ms        77.2 ms         n/a
Bybit ETHUSDT tick sample       497.7 ms        143.3 ms        n/a
Grimoire Bybit 900s book rows   141.7 ms        49.2 ms         77.9 ms
```

## Findings

For candle-like rows, this historical probe let the writer choose
open/high/low/close shape programs from parent relationships alone:

```text
ts      implicit fixed step
open    previous close offset
high    max(open, close) residual
low     min(open, close) residual
close   open-relative offset
volume  base-offset bitpack
```

That behavior is no longer the schema contract. Parent bytes authorize direct
related deltas only. The current header dialect uses `101-199` derived
expression refs for min/max residual calculations, with the operation and input
slots defined in the schema header. The equivalent current map for the same
shape would mark the min/max-derived output slots with expression refs:

```text
100 0 102 103 2 0

expr2: output slot 2 = max(slot 1, slot 4) + residual
expr3: output slot 3 = min(slot 1, slot 4) - residual
```

For ticks, the plain root map is enough. The useful transforms are previous-row
or base-offset bitpacks, not same-row parent relationships. The UUID was split
into two i64 halves for this probe; stats must not attempt signed previous
deltas when adjacent opaque halves span more than the signed i64 delta range.
That overflow case is covered by a regression test.

For Grimoire-style orderbook rows, the old `128` repeated-child scope proved
that orderbook data needs explicit grouping, not hundreds of flat level slots.
In the current v1 dialect that idea is represented with group bytes `201-239`
and, for bid/ask structures, the scoped dual-domain wrapper `200`. The planner
must score those relationships instead of forcing related deltas. On the
parented grouped run, Aura0 stamped generic partition run lengths, optional
runs-per-event, grouped event-value streams, partition-based first price
offsets, inside-run price deltas, a multi-slot sparse presence map, packed
dictionaries for nonzero quantity streams, and a presence-derived flag. That
reduced the parented-header `.aura0` from the earlier 794558-byte bad-delta
result to 276998 bytes on the same decoded rows, with ingest, Aura0, and Aura1
decode checks all matching the normalized rows.

## Caveats

The Bybit tick run used the first 50000 trades from each public archive day
2026-06-09 through 2026-06-13. The full five daily archives are about 421 MB
compressed upstream, and the current generic i64 writer is still all-memory.
That is a writer scalability limitation, not a schema-header ambiguity.

The generic planner now proves the header relationship is clear enough and is
part of the `.aura -> .aura0` conversion path. Ingest stamps the generic Aura0
plan into the `.aura` footer. Conversion writes an Aura0 body from that stamped
plan and preserves the same plan in the compiled footer so decode does not need
Rust-side re-inference.
