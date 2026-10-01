# Arrival collections — an array in the envelope becomes the composition children of the created record

- **Status:** draft
- **Issue:** [eclipse-dirigible/dirigible#7593](https://github.com/eclipse-dirigible/dirigible/issues/7593)
- **Implementation:** [eclipse-dirigible/dirigible#7594](https://github.com/eclipse-dirigible/dirigible/pull/7594) (released in [14.70.0](https://github.com/eclipse-dirigible/dirigible/releases/tag/v14.70.0))
- **Companion:** [`0021-arrival-mapping.md`](0021-arrival-mapping.md) — the envelope reading this one
  extends from single values to a set.
- **Discussion:** (this PR)

## The problem

[`0021-arrival-mapping.md`](0021-arrival-mapping.md) lets an arrival read an envelope: a key becomes a
field, a business key becomes a relation. Every value it can name is a **single** value. A real
contract regularly carries a **set**:

```json
{ "orderNumber": "PO-1042", "customerCode": "C-7",
  "lines": [ { "sku": "A-100", "qty": 2 }, { "sku": "B-200", "qty": 1 } ],
  "tags":  [ "rush", "gift" ] }
```

The model already has the right home for it — `Order` owns its `OrderLine`s through a composition —
but `map` cannot reach it: a value is an envelope key or a lookup, and an array is neither. The routes
left are each worse than the problem:

1. **Hand-written code** for the whole arrival, because one array in an otherwise declarable envelope
   is enough to lose the construct — the outcome the envelope reading was written to prevent.
2. **One message per element.** The sender splits the set, and the receiver creates the master on the
   first message and the children as they trickle in. Nothing marks the set complete, so whatever the
   master's creation starts — a process, a notification — runs against a partial order, and a failure
   in the middle leaves half of one stored.
3. **The array as text.** Store it in a scalar field and split it later in hand-written code. That puts
   a set into a column, which the model otherwise never does, and the children exist only after code
   nobody declared has run.

The second deserves the emphasis. A receiver's reactions are keyed on the master's creation, and a
master created before its children is a master every reaction observes incomplete.

## The proposed shape

A `map` key that names a one-to-many relation of `create` whose target is a **composition child** of it
takes a **collection**: where the array is, and how one element becomes one child.

```yaml
inbound:
  - name: orders
    source: { queue: orders }
    accept: { type: order.placed, version: 1 }
    create: Order
    map:
      orderNumber: orderNumber
      customer: { lookup: Customer, by: code, from: customerCode }
      lines:                                   # Order.lines -> OrderLine, a composition child of Order
        from: lines                            # the envelope key holding the array
        max: 100                               # optional: an upper bound on the elements
        map:
          quantity: qty                        # child field <- element key
          product:  { lookup: Product, by: sku, from: sku }   # a lookup per element
      tags:                                    # an array of plain values
        from: tags
        map: { label: "." }                    # "." is the element itself
```

A `map` without a collection keeps its current meaning exactly; a file that declares none changes
nothing.

## Expected behaviour

**Everything is read before anything is stored.** Every element of every collection is resolved — its
shape checked, its values converted, its lookups answered — before the arrival stores anything.

**One bad element rejects the whole arrival.** Any of these rejects the arrival exactly as an
unresolved lookup does today, with a diagnostic naming the collection and the position of the element:

- the collection's key holds something other than an array;
- an element is null, or of the wrong shape — an object where element keys are declared, a plain value
  where `"."` is;
- the array holds more elements than `max`;
- an element lacks the key one of its lookups reads;
- an element lookup matches no record.

Nothing of a rejected arrival is stored — not the master, and not the children that did resolve.

**The master and its children are stored as one atomic unit.** Either all of them are stored or none
is, and nothing outside the arrival observes the master without its children. Whatever reacts to the
master's creation — a process it starts, a notification it sends — finds every child already there,
and one arrival starts one such reaction, however many elements it carried. A child the store refuses
(a uniqueness rule, a required value) fails the whole unit.

**An absent or empty array means no children.** It is not an error: an order may arrive without tags.

**Elements are kept as sent.** The construct takes no position on duplicates or on order. If two equal
elements must not both be stored, the child says so with a uniqueness rule, which refuses the arrival
like any other refused child.

**Children are written like any other record.** Each child is stored through its entity's ordinary
write path, with its back-reference to the master filled from the master just created. An element key
that is absent leaves its property unset, which the child's required-value rules then judge.

## Edge rules

- **Only a composition child.** The key MUST name a one-to-many relation of `create` whose target is a
  composition child of `create`. The composition is what makes the children the master's own — deleted
  with it, locked with it. A one-to-many whose rows have a life of their own relates independent
  records, and an arrival does not create those.
- **Same model.** The child is an entity declared in the same model, as the lookup target of
  `0021-arrival-mapping.md` is.
- **The collection's keys are closed:** `from` and `map`, both required, and `max`. An unrecognised key
  is an error (per [`0014`](0014-unknown-keys-are-errors.md)), and so is `from: "."` — the envelope
  itself is not an array.
- **`max` is a whole number from 1 to 10 000.** A larger bound is refused: an arrival that legitimately
  carries more children than that is a bulk import, which a message is not.
- **An element key names a field or a to-one relation of the child** — the rule
  `0021-arrival-mapping.md` sets for the master.
- **Never the child's key, never its back-reference.** The key is assigned when the child is created,
  and the back-reference is filled from the master. Mapping either is an authoring error.
- **`"."` is the element itself.** It serves an array of plain values, as a field's value or as a
  lookup's `from`. A collection that uses it MUST NOT also name element keys — an element is either a
  value or an object.
- **Element lookups follow the lookup rules of `0021-arrival-mapping.md` unchanged** — `by` identifies
  at most one record and is text or an integer — with `from` an element key or `"."`.
- **One collection per child entity.** Two collections filling the same child would fill one set of
  rows twice.
- **No nested collections in this revision.** An element whose own array becomes grandchildren has no
  motivating case yet, so a collection inside a collection's `map` is an error until one appears.
- **A valueless key is an error**, as for the master: `map: { lines: { from: lines, map: { quantity: } } }`
  is reported rather than read as filling nothing.

## Prior art / workarounds

The three routes in *The problem*: a hand-written consumer, one message per element, or the array as
text. Each is in use, and each moves the set out of the model — into code, into the sender's protocol,
or into a column.

Considered and not proposed:

- **A subset-style scalar** — the element values kept as one list-valued field of the master. It fits
  plain values only, loses every per-element field and lookup, and gives the children no rows of their
  own to be validated, reported on or related to.
- **One message per element, made safe** with a count or an end-of-set marker. It moves an atomicity
  problem into a protocol both sides must implement, which is the problem this proposal removes.
- **Nested collections now.** No contract seen so far needs grandchildren in one arrival; specifying
  them before one does would be guessing at the rules.
- **A general transformation language.** The argument of `0021-arrival-mapping.md` holds unchanged: a
  value vocabulary of keys, `"."` and lookups covers the contracts people have, and an arrival that needs
  more is an algorithm.

## Specification text

**Anchor:** Declarative glue > `inbound — arrivals from outside` > a new subsection immediately after
*Reading an arrival as an envelope — `accept` / `map`*.

It also widens one sentence of that subsection's normative block. The sentence

> Each key of `map` MUST name a field or a to-one relation of `create`;

becomes

> Each key of `map` MUST name a field or a to-one relation of `create`, or — as a collection, below — a
> one-to-many relation of `create` whose target is a composition child of it;

The new subsection:

#### An envelope array as composition children — collections

An envelope often carries a set: an order with its lines, a request with the items it applies to. A
`map` key naming a one-to-many relation of `create` whose target is a composition child takes a
**collection**, and the array becomes the master's children — one child per element, stored with the
master as one unit.

```yaml
inbound:
  - name: orders
    source: { queue: orders }
    create: Order
    map:
      orderNumber: orderNumber
      lines:
        from: lines
        max: 100
        map:
          quantity: qty
          product:  { lookup: Product, by: sku, from: sku }
      tags: { from: tags, map: { label: "." } }
```

`from` names the envelope key holding the array, `map` fills one child from one element, and `"."` is
the element itself, for an array of plain values.

> **Normative.**
> A collection declares exactly `from` and `map`, and optionally `max`. Its key MUST name a one-to-many
> relation of `create` whose target is a composition child of `create`, declared in the same model; at
> most one collection of an arrival fills a given child entity. `from` names an envelope key and MUST
> NOT be `"."`. `max`, when declared, is a whole number from 1 to 10 000.
>
> Each key of the collection's `map` MUST name a field or a to-one relation of the child, and MUST NOT
> name the child's key or its back-reference to the master, which is filled from the master the arrival
> creates. A value is an element key, `"."` — the element itself — or a lookup as above, whose `from` is
> an element key or `"."`. A collection MUST NOT combine `"."` with element keys, and MUST NOT contain a
> collection.
>
> Every element MUST be resolved before anything of the arrival is stored. The arrival MUST be rejected,
> with a diagnostic naming the collection and the element's position, when the key holds anything but an
> array, when an element is null or not of the declared shape (an object for element keys, a plain value
> for `"."`), when the array holds more than `max` elements, when an element lacks the key a lookup
> reads, or when an element lookup matches no record. A rejected arrival stores nothing.
>
> An absent or empty array creates no children. Elements are kept as sent: the construct removes no
> duplicate and promises no order.
>
> The master and its children MUST be stored as one atomic unit, each through its entity's ordinary
> write path: either every one of them is stored or none is, and a reaction to the master's creation
> MUST observe all of its children. A child the store refuses fails the whole arrival.

## DSL index

**Anchor:** Appendix A: DSL index, immediately after the `inbound.accept` / `inbound.map` row.

| Construct | What it gives you |
| --- | --- |
| [`inbound.map` collections](#an-envelope-array-as-composition-children--collections) | an envelope array becomes the composition children of the created record, stored with it as one unit |
