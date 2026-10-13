# Province modifier lifecycle

A province stores linked nodes pointing to active-modifier records. Each record
holds a shared definition pointer and a date value. Removing a modifier can
unlink its node immediately or only mark it for later removal. Marking is not
physical removal and does not itself make the presence trigger ignore the node.

Addresses are preferred x86 **VAs** (image base `0x00400000`); offsets are from
complete objects. The province is named `CMapProvince` in this repository.
The layouts below cover selected fields, not a complete class or ownership model.

## Definition lookup

The add/remove effect and presence-trigger parsers use the definition registry
at `0x0125A69C` and name lookup `0x005661B0`. Lookup takes the registry in EAX,
one stack MSVC-string pointer, and returns the definition in EAX.

| Parser lookup callsite | Retained definition |
| --- | --- |
| `0x00881337`, add effect | `this+0x24`, stored at `0x0088134C` |
| `0x0088217A`, remove effect | `this+0x20`, stored at `0x0088218F` |
| `0x008CDEAA`, presence trigger | `this+0x1C`, stored at `0x008CDEBF` |

These are instruction addresses inside parsers, not function entries. Registry
resolution has selected active-campaign correlations; complete registry identity,
duplicate-name behavior and cross-session pointer lifetime remain unresolved.

## Storage

| Object | Offset | Meaning |
| --- | --- | --- |
| Province | `+0xC8` | Modifier-list head node |
| Province | `+0xCC` | Tail node |
| Province | `+0xD0` | Linked-node count |
| Province | `+0xD4` | Byte guard: nonzero defers physical unlinking |
| List node | `+0x00` | Active-record pointer |
| List node | `+0x04/+0x08` | Previous/next node |
| List node | `+0x0C` | Byte deferred-removal marker |
| Active record | `+0x08` | Definition pointer |
| Active record | `+0x0C` | Date value used by addition and expiry paths |

Nodes and records are **separate `0x10` allocations**. Names for these two
structures are descriptive; their complete native class identities and trailing
node bytes are not established.
The count includes marked nodes until physical unlinking.

## Addition

Native path `0x004B3A30` computes a date, allocates a record at
`0x004B3ACF/0x004B3AD1`, and looks up the definition by name at `0x004B3B25`.
It writes the definition and date at `0x004B3B3E/0x004B3B41`.

A separate node allocation at `0x004B3B55/0x004B3B57` stores that record, the old
tail, a null next pointer and marker zero (`0x004B3B63`–`0x004B3B6F`). Append
updates the tail/count, links the previous tail or initializes the head
(`0x004B3B77`–`0x004B3B95`).

These mechanics are statically established. Live addition, duplicate handling,
exact duration/sentinel semantics and expiry encoding are not established here.

## Presence

`CHasProvinceModifierTrigger::Evaluate` (`0x008CDFC0`, vtable `0x00E30890`
slot 6 / `+0x18`) reads the province **index** from scope `+0x10` and resolves it
through `[0x012588E8]+0xACC`, the world province-pointer array.

It walks province `+0xC8`, follows node `+0x08`, and compares record `+0x08` with
trigger `this+0x1C` (`0x008CDFE4`–`0x008CDFEC`). A match returns true; exhaustion
returns false. It does not inspect the node marker or record date. A linked,
valid record marked by the remove effect can therefore still report present.

## Removal

`CRemoveProvinceModifierEffect::Execute` (`0x00882290`) executes through virtual
slot 11 / `+0x2C`, with ECX=effect and two stack arguments (`ret 8`). It resolves
the scope's province index, loads definition `this+0x20`, and calls remover
`0x004B39A0` at `0x008822AC` with ESI=province and one stack definition pointer.
The remover returns `ret 4`, with no success-result contract.

| Province guard `+0xD4` | Effect on a matching node | Primary instructions |
| --- | --- | --- |
| Nonzero | Set node marker to 1; leave links and count unchanged | `0x004B39D7`–`0x004B39E4` |
| Zero | Repair neighbors/head/tail, free node and decrement count | `0x004B39E6`–`0x004B3A1F` |

The outer walk saves the next node before removal and continues through duplicate
matches. These branches are statically established. Immediate removal and list
postconditions have narrow runtime correlations; duplicate removal and deferred
completion are not runtime-established.

## Expiry and record lifetime

The reviewed expiry path reads record date `+0x0C` at `0x004B3750` and uses the
same guard/marker/unlink pattern. It also calls the record's virtual deleting
destructor with flag 1 (`0x004B381E`–`0x004B3826`), including on its mark-only
branch (`0x004B37CC`–`0x004B37D8`). The effect remover `0x004B39A0`, by contrast,
frees matching list nodes without a record-destructor call in that body.

Node unlinking, record destruction and shared-definition lifetime are different
operations. This distinction does not prove a memory leak or establish safe
retention of borrowed pointers across cleanup.

## Limits

- The province's deferred-drain callsite and timing remain unidentified. Generic
  list-drain code at `0x005499C0` is not established as this province's drain.
- Date/sentinel semantics, full record ownership and save interaction remain
  unresolved; the reviewed expiry code is not a complete expiry specification.
- Layout and callsite evidence do not authorize mutation from arbitrary threads
  or callbacks. Runtime evidence covers selected synchronous operations, not
  every lifecycle boundary.
