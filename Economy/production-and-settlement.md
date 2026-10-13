# Production and settlement accounting

Requested inputs, physical stock, production and sales proceeds are different quantities.
This page follows bounded factory, artisan and RGO paths in `v2game.exe` 3.04, not a complete ledger or layout.

Addresses are preferred-base VAs (image base `0x00400000`), not RVAs. **[C]** means
confirmed in native code; **[M]** means a narrow runtime/save correlation, not a
general contract. See [economy-daily-path.md](economy-daily-path.md) for scheduling
and [stockpiles.md](stockpiles.md) for the goods-pool representation.

## Factory requests are not purchases **[C]**

Offsets below are relative to the complete `CStateBuilding` object.

| Offset / boundary | Meaning |
|---|---|
| `+0x28` | Input stock goods pool; a balance, not today's actual purchases |
| `+0x80` | Requested-input goods pool; not fulfilled delivery |
| `+0xD8` | Produced output quantity used by the sales caller |
| `+0x160/+0x164` | Sales amount assigned by `0x004F4B30` |
| `+0x1F8/+0x1FC` | Unsold output inventory carried through sales settlement |

`0x004DD470` builds a request, in this order:

1. Copy recipe inputs from `CProductionType+0x24` into a destination pool.
2. Scale by the supplied fixed-point factors and integer factory level.
3. Add scaled maintenance inputs from the recipe pool at `+0x90`.
4. Subtract supplied input stock via `0x0047DCA0` at `0x004DD5AD`.
5. Clamp negative requirements to zero and apply the upper bound via `0x0047D9E0`.

The subtraction changes the **request destination**, not the original stock.
`0x004F3566` selects factory `+0x80` for request storage. The daily path calls
`0x004DD470` at `0x00484573`, then copies the good-index map and quantities there.
Stock already on hand reduces requested demand; shortages can leave requests
unfulfilled. Requested input value need not equal spending. **[M]**

Actual production is separate: `0x004816B0 -> 0x004F3620` supplies factory `+0x28`
to `0x004DDE00`. Scaled recipe inputs are subtracted at `0x004DE0E3/0x004DE0EA`;
`0x004DE0FA/0x004DE0FD` store residual stock. An alternate branch uses a temporary
pool, so this is not a claim that every production path consumes identically.

## Factory physical boundaries **[C]**

Observe stock **before** the call at `0x004845FD -> 0x00482FF0`
(`Factory_MarketUpdate`). The factory is the third stack argument. This is the
pre-purchase, post-consumption boundary on the identified daily path.

Inside settlement, `0x0048368E` selects factory `+0x28`. Calls at
`0x0048369D` and `0x004836AA` pass the two delivered pools to `0x0047DC20`.
Its `0x0047DC59/0x0047DC5F` use `ADD/ADC` to add quantities to destination stock.
These additions, not the request pool, are actual delivery on this path.

For each good, define `B_t` as pre-purchase stock, `D1_t/D2_t` as deliveries,
and `C_t` as stock after both additions:

```text
C_t = B_t + D1_t + D2_t
consumed_t = C_(t-1) - B_t                 [only with continuity and no other transfers]
```

The first equation covers the identified additions. The second needs a complete
previous closing boundary and a continuous stock account. More generally:

```text
B_t = C_(t-1) - consumed_t + net_other_transfers_t
consumed_t = C_(t-1) + net_other_transfers_t - B_t
```

Here `net_other_transfers` is net inflow outside the accounted deliveries/consumption.
An unexplained difference is not automatically consumption. Skipped purchase calls,
closed factories and missing phases are not zero consumption or complete intervals;
carry stock forward only when the intervening path is accounted for.

Quantities use 15-bit fixed point (`raw / 32768`). Spending plus opening stock value
minus closing stock value is only a monetary estimate: transaction prices and the
prices used to value inventory may differ. Physical stock differences avoid that
valuation assumption, but still require continuity.

## Factory sales: inventory and proceeds **[C]**

`0x00488710 -> 0x004F4B30` passes the live factory first, then a 64-bit sales amount
(callee `[EBP+0x10/+0x14]`). `0x004F4B59/0x004F4B5F` **assign**, not accumulate,
that exact amount to factory `+0x160/+0x164`.

The caller reads produced quantity at `+0xD8` and opening unsold quantity at `+0x1F8`.
It retains opening unsold in caller locals `+0xA0/+0xA4` and writes closing unsold at
`0x00488489/0x00488493`. For the same output good and a complete interval:

```text
sold = opening_unsold + produced - closing_unsold
proceeds = amount passed at 0x00488710
```

The quantity equation assumes no unaccounted output transfers. Inventory carryover
means sold quantity need not equal today's production. Proceeds are observed
independently; neither `output * price` nor a sold/output ratio replaces them.

Budget treatment depends on the opening balance:

| Opening budget | Native treatment |
|---|---|
| Nonnegative | Add the sales amount at `0x004F4C2D/0x004F4C36`; replace a negative result with `INT64_MAX` at `0x004F4C4E/0x004F4C58`. |
| Negative | Clear the budget at `0x004F4C6A/0x004F4C70`, then add the same sales amount at `0x004F4C76/0x004F4C7C`. |

Both branches apply the sales amount; the condition selects budget treatment,
not whether sales are credited. Resetting a negative balance and clamping mean
sales assignment does not establish a universal budget-delta identity or
explain every subsequent funding and payment path.

## Artisan stock versus requirement **[C]**

The artisan economy pointer is read at `CPop+0x1D4`. Relative to that economy object,
`+0x00` is stock and `+0x58` is the requested recipe-input pool.

`0x00486B81 -> 0x004DD3C0` copies/scales recipe inputs into a temporary requirement
pool, then calls `0x0047DCA0` at `0x004DD43E` to subtract stock **from that pool**.
It does not subtract requirements from actual artisan stock. This call is therefore
not a physical-consumption boundary, even though the helper's older name mentions
artisan/factory stockpile subtraction.

Settlement is `0x00486BFF -> 0x00483AA0`, with the POP in `EBX` at the caller.
Actual delivery additions are `0x00483FCA/0x00483FDA -> 0x0047DC20`, with the POP
in `EDI` and its economy object in `ESI`. Apply the physical equations above only
with complete pre-purchase/closing observations; request generation cannot supply
the missing consumption phase.

Scaled recipe consumption has matched previous closing stock minus residual
stock on complete, continuous intervals. **[M]** Recipe changes and incomplete
settlements break that inference. Output sales likewise need continuous
opening/closing inventory for the same output good, with proceeds kept separate.

## RGO registry and output **[C]**

`0x004C25B0` is the lazy registry accessor using global `0x01258728`; it does not
itself index a province. The caller reads province ID at `0x00740ED6`, multiplies
it by `0xB0` at `0x00740EDE`, then adds the registry's record-array base.
Initialization writes province pointers at record `+0x1C` and advances by `0xB0`
at `0x004C2621/0x004C2626`.

The **complete per-province record stride is `0xB0`**; embedded `CStateEmployment` starts
at record `+0x10`, so `0xB0` is not its size. Relevant fields:

| Record offset | Meaning |
|---|---|
| `+0x08` | Production-type pointer |
| `+0x1C` | Province pointer |
| `+0x20/+0x24` | Employment-assignment vector begin/end |
| `+0x38` | Efficiency/output multiplier, `/32768` |
| `+0x40` | Throughput multiplier, `/32768` |
| `+0x58` | Assigned worker total |
| `+0x88` | Size multiplier, `/32768` |

Assignment entries are 16 bytes; `0x004ED776/0x004ED77A` read and sum their `+0x0C` counts.
The destination selected at `0x004EE535`, employment `+0x48`, is record `+0x58`.

`0x004C11CF` passes record `+0x10` to output helper `0x004DE350`, so its employment
reads at `+0x28/+0x30` are record `+0x38/+0x40`. With raw fixed-point operands:

```text
q1 = (efficiency * throughput) >> 15
q2 = (q1 * CProductionType[+0x88]) >> 15
output = (q2 * record_size[+0x88]) >> 15
```

Shifts occur **after each multiply**, at `0x004DE36E/0x004DE384/0x004DE3A4`, not
once after a combined product. Preserve native 64-bit overflow behavior. The caller
also has null/sentinel branches that return zero; the formula is the valid path.

## Limits

These callsites establish a bounded subset, not a universal layout, registry count
or arbitrary-mod contract. Alternate, invalid and skipped paths need their own accounting.
Observed boundaries do not establish completeness of unobserved intervals.
Neither these stock equations nor aggregate market sold totals establish global GDP,
per-producer domestic/export attribution, or cash creation/conservation. An RGO income
field is not proof of newly created cash. Keep physical flows, inventory valuation and
money transfers separate until their respective boundaries are established.
