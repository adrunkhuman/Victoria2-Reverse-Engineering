# PART V — GOODS-POOL SUBTRACTION

## MAP 13 — `SubtractArtisanFactoryStockpileFromESIStockpile` — 0x0047DCA0

This is a generic goods-pool subtraction helper. The stack argument supplies the source pool;
`ESI` supplies the destination pool. It changes the destination, not the source. Its historical
name does not make every call an artisan/factory stock-consumption operation.

Factory and artisan request generation subtract stock from a temporary requirement pool;
actual stock remains unchanged at those calls. See
[production-and-settlement.md](production-and-settlement.md) for physical boundaries and
[economy-daily-path.md](economy-daily-path.md) for the daily call path.

```text
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│         SubtractArtisanFactoryStockpileFromESIStockpile  (0x0047dca0)                        │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

For each good:

source_pool.good_indices[good]
                │
                ▼
          Is good present?
                │
          ┌─────┴─────┐
          │           │
         NO          YES
          │           │
          ▼           ▼
        SKIP    Check destination ESI
                         │
                  ┌──────┴──────┐
                  │             │
              absent          present
                  │             │
                  ▼             ▼
           create entry    subtract int64
           with -amount    directly
```

Important fields:

```text
[0x12587f4]
    number_of_goods

[pool + 0x08 + good]
    quantity-slot index (zero = absent)

[pool + 0x48]
    quantity vector begin

[pool + 0x4C]
    quantity vector end
```

Stockpile entries are:

```text
8 bytes
=
64-bit integer
=
2 × int32
```

The vector size is obtained through:

```text
(end - begin) >> 3
```

When the destination has no entry:

```text
source_pool amount
        ↓
NEG 64-bit value
        ↓
push_back into ESI stockpile vector
```

When the destination already has an entry:

```text
ESI stockpile
        -
source_pool quantity
        ↓
64-bit SUB + SBB
```

Therefore the conceptual result is:

```text
ESI.stockpile[good]
    -=
source_pool.quantity[good]
```

### Important finding

The byte at:

```text
pool + 0x08 + good_index
```

maps the good ordinal to a slot in the quantity vector; zero means absent.
The amount is not necessarily at `vector[good_index]`.

The actual amount is stored in the stockpile vector at:

```text
pool + 0x48
```

with 8-byte entries.

This is important for understanding the goods representation used by the market system.
