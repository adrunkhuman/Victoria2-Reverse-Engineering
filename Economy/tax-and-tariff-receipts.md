# Tax and tariff receipts

In Victoria II: Heart of Darkness 3.04 (English, x86), class-tax income is summed
and converted; positive tariffs credit treasury during trade settlement. Neither explains all treasury movement.
Addresses are preferred-base VAs (`0x00400000`); relocate them for runtime use.
See [the daily economy path](economy-daily-path.md) for market context.

## Evidence and scope

| Mark | Applies to |
|---|---|
| **[C]** | The specific native instructions, fields and callsites below, not whole object layouts. |
| **[M]** | Narrow runtime/save correlations of class-tax receipts, ledger entries and scale. |
| **[?]** | Runtime tariff amount correlation and the complete negative-rate/subsidy contract. |

Treasury is signed 48.15 fixed point: pounds = `treasury_raw / 32768`.

## Class identity and pointer indirection **[C]**

`CCountry+0xA0` holds the vector backing-storage pointer. Its first three
four-byte slots hold **tax-object pointers**, not rates or income amounts:

```text
slots = read_ptr(country + 0xA0)
poor_tax   = read_ptr(slots + 0x00)
middle_tax = read_ptr(slots + 0x04)
rich_tax   = read_ptr(slots + 0x08)
```

At `0x00508C8D/93`, `mov ecx,[ebx+0xA0]; mov edx,[ecx]` demonstrates both
loads for poor tax. Middle and rich repeat them at `0x00508CCB/D1` and
`0x00508D07/0D`. **Do not read country `+0xA0/+0xA4/+0xA8` as three sliders.**
The existing Cheat Engine layout notes identify those country fields as vector
begin/end/capacity; the class slots are in the separate backing allocation.

| Class | Backing slot | Save name | Token | Token registration VA |
|---|---|---|---|---|
| Poor | `+0x00` | `poor_tax` | `0x874` | `0x00C4F248` |
| Middle | `+0x04` | `middle_tax` | `0x875` | `0x00C4F274` |
| Rich | `+0x08` | `rich_tax` | `0x876` | `0x00C4F2A0` |

The country save writer at `0x005004B7–0x0050050E` follows these pointers,
emits rich/middle/poor tokens, and calls each object's virtual save method.
This identifies the classes; it does not reconstruct all tax-setting fields.

## Aggregate conversion, once per class **[C]**

The three calls at `0x00508C9B`, `0x00508CD5` and `0x00508D11` invoke
`FUN_00439630(tax_object, output_pair)`. It reads the income-array pointer at
`tax_object+0x18`, sums 64-bit entries with `ADD/ADC`, and clears each consumed
entry (`0x004396C4–0x0043972F`). The entry count comes from the POP-type registry,
not from a three-element income array. Only after summing does it convert.

`0x00439741/47` load the **contents** at `0x00F0AACC` and `0x00F0AAC8`:
the high/low words of the divisor `32768000 = 1000 << 15`, not a pointer to
another divisor. `SHLD/SHL` at `0x0043974E/53` shift the aggregate left 15;
`0x00439758` calls signed 64-bit division (`__alldiv`, `0x00AC02A0`). The result
is written to the caller's output pair at `0x00439760/62`.

```text
receipt_raw = trunc_toward_zero((sum_native_income << 15) / 32768000)
receipt_raw = floor(sum(serialized tax_income) * 32768 / 1000)  # nonnegative
```

The second form uses the save-scale correlation: serialized income
is native income divided by `32768`, and its monetary scale is `/1000` **[M]**.
It is not a per-POP floor. For nonnegative, non-overflowing totals the rounding
loss is less than one treasury-raw unit per class. Native 64-bit overflow and
signed division still apply; a floor claim is not extended to negative totals.

## Treasury and selected daily accounting **[C]**

Each returned low/high pair is added to country treasury `+0xE78/+0xE7C`
and, unchanged, to the selected daily-accounting entry:

```text
ledger = read_ptr(country + 16 * (read_u32(0x01258738) + 13))
```

For example, `0x00508CB4–0x00508CBE` load the global index, add 13, double it,
and dereference `[ebx+eax*8]`. The following offsets are relative to that
**selected entry pointer**, not inline country fields or the whole ledger layout.

| Class | Treasury `ADD / ADC` VAs | Ledger `ADD / ADC` VAs | Entry low/high offsets |
|---|---|---|---|
| Poor | `0x00508CA4 / 0x00508CAE` | `0x00508CC1 / 0x00508CC8` | `+0x00/+0x04` |
| Middle | `0x00508CDE / 0x00508CEC` | `0x00508D00 / 0x00508D04` | `+0x08/+0x0C` |
| Rich | `0x00508D1A / 0x00508D25` | `0x00508D38 / 0x00508D3B` | `+0x10/+0x14` |

## Positive tariffs **[C]**, runtime amount **[?]**

Trade settlement reads the setting pair at `CCountry+0x1440/+0x1444`
(`0x00488A58/62`). On the positive-rate path, `FUN_005339D0` supplies a country
factor; `__allmul` calls at `0x00488AB9` and `0x00488AD1` multiply factor ×
setting, then the computed trade amount. Each multiply is followed by a signed
15-bit right shift (`0x00488AC7/CC`, `0x00488AD6/DA`), with native 64-bit overflow.
The final pair credits treasury at `0x00488ADD/0x00488AE3` and the same selected
ledger's `+0x18/+0x1C` at `0x00488AF7/0x00488AFA`.

The negative-rate branch instead calls the money sink `FUN_00525770` at
`0x00488B9C`. That callsite does not establish the full subsidy contract.
Runtime tariff amount correlation remains unresolved; the positive receipt is
supported by static callsites, not by a claimed runtime amount match.

## Limits

These receipts do not establish the full POP tax calculation, AI slider policy,
full scheduling or ledger layout, or arbitrary-executable/mod compatibility.
They are not a complete treasury balance or money-conservation account: gold,
other sources/sinks and write-off behavior remain outside this contribution.
