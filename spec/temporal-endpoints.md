# Scalar temporal endpoint observations

Status: END-1 through END-10 are agreed, but for the sentences of END-7,
END-9 and END-10 that begin "Proposed", which are proposed.

## Scope and API

This slice covers owned scalar outputs and fixed, directly bound scalar
inputs during evaluation. Nullable scalar values are included. END-7 gives
the validity of a collection, END-8 the read of its child, END-9 a window's
readiness and what it holds, and END-10 when a reference ticks. Other collection
observations, the rest of a reference's contract, sampled/rebound inputs,
and source scheduling have their separate contracts. The [language temporal model](https://github.com/hhenson/hgl_spec/blob/main/spec/language/temporal-model.md)
owns endpoint interpretation, handler selection, and phase admission.

| HGL observation | Runtime result / admission |
| --- | --- |
| `valid(value)` | bool; may inspect an invalid endpoint; for a collection, see END-7 |
| `all_valid(value)` | bool; a list, tuple, struct, dictionary or window only; see END-7 and END-9 |
| `modified(value)` | bool; whether the endpoint published in the current cycle, but for an input of a branch as it starts (END-2) and a window of a duration at an exit (END-9); a guard that admits a payload read as `valid` does |
| Runtime parameter payload | T; requires readable valid input, subject to declared nullable/default argument binding |
| A collection's child, `parts[1]`, `quote.bid` | An endpoint, valid or not; its payload requires its own admission; see END-8 |
| A `ref` parameter, `ref low: i64`, and its children | Endpoints, not values: no payload or `delta_value` read (the language's FN-16); see END-10 |
| `delta_value(value)` | `delta<T>` (the language's TYPE-16): T for a scalar input, and for a nullable scalar the optional that holds what was published; see END-3 |
| `is_active(input)` | bool; current activity of this direct input, including while invalid |
| `last_modified(value)` | Agreed `datetime?`; see END-6 |

`valid`, `all_valid`, `modified`, `delta_value`, `last_modified`, `invalidate`,
`is_active`, `activate`, and `passivate` are declared in `hgl/native/endpoints.hgl` as
native value functions that take the endpoint and not its payload, an
`endpoint<T>` (the language's TIME-9). Runtime metadata reads create no graph
nodes. Composition forms and empty/multi-argument handler
selectors follow the language contract rather than this one-endpoint table.

## Agreed rules

- **END-1.** A scalar publication makes the endpoint valid and retains its
  payload, including false, zero, empty text, null, NaN, or infinity.
  Silence preserves validity and the retained payload.
- **END-2.** A scalar publication marks its endpoint modified for the current
  cycle, including when the payload repeats. An idle cycle leaves the endpoint
  unmodified and preserves its retained payload. A fixed, directly bound input
  observes its output's state. One exception is an input of a node in a
  branch started while the program runs (the language's CTRL-2): bound to
  an endpoint that is already valid, it is modified in the cycle the branch
  starts, published in that cycle or not, and wakes its node as a
  publication does, so the branch's nodes run once with the values held.
  Other inputs bound to that endpoint see it as it is. The other is a
  window of a duration, modified at an exit with nothing published
  (END-9). An input is not
  modified without being valid: `modified(x)`, as a guard, admits a read
  of `x` as `valid(x)` does (the language's TIME-4). The
  [conditional cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/conditional_graphs.hgl)
  check a branch's start, and the
  [endpoint cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/scalar_endpoints.hgl)
  a read under `modified`.
- **END-3.** `delta_value(x)` returns what `x` published this cycle, a
  `delta<T>` for an endpoint of T (the language's TYPE-16). For a scalar it
  is the value published. For a nullable scalar it is the optional that
  type gives it, which holds the value published or the null: a published
  null is `delta<i64?>()` for an `i64?` endpoint, and is no absent delta.
  For a list, the delta read gives the new size where it moves, never a
  negative one, and on a shrink names the dropped tail positions in
  `removed`, also where the delta written gave only a smaller
  `size_modified`, or a negative one, which reduces the list by that
  count and leaves it empty where the count is more than it holds. An
  `updated` entry written at the position after a `list<V>`'s last, with
  no `size_modified`, extends the list. An entry past a `list<V, N>`'s
  bounds in a delta made as the program runs fails the write that
  publishes it (the language's TYPE-16). An entry written further past a
  `list<V>`'s last position extends the list as well, and the positions
  between are not valid. An `updated` entry at or past the size that a
  `size_modified` in the same delta gives contradicts it, and the write
  that publishes it fails. A `removed` that names a key a dictionary does
  not hold, a member a set does not hold, or a position of a list that
  holds no value, never set or already removed, is ignored, and is no
  failure. A delta that names no change, `delta<set<i64>>()`,
  `delta<dict<str, i64>>()` or `delta<list<i64>>()`, ticks only where it
  changes validity (the language's TIME-7). Where a `list<V>`, a
  dictionary or a set is not valid, never published or invalidated
  (END-4), it makes the endpoint valid, empty and modified, and
  `delta_value` returns that delta. Where the endpoint is valid it is no
  publication (END-5). Such a delta of a struct, a tuple or a
  `list<V, N>` sets no child and changes no validity (END-7): it is never
  a tick. A delta whose every entry is a removal ignored as above names no
  change too, and ticks only where it makes the endpoint valid. The
  [collection cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/collection_ticks.hgl)
  check a list and a set, and the
  [dictionary cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/dictionary_ticks.hgl)
  a dictionary.
  The checker must prove `x` is valid and modified at the call.
- **END-4.** Invalidation clears validity, modification, payload, and delta.
  The transition to invalid notifies active subscribers. A node awakened by
  it can observe the invalid input in `when valid([])`. An explicit
  `modified(input)` guard is false on that transition, even on an active input. Repeated invalidation while invalid is
  silent. Within a producer evaluation the last action determines the state:
  publish then invalidate leaves the endpoint invalid. Invalidate then publish
  leaves it valid and modified.
- **END-5.** A publication notifies active subscribers, including when its
  value repeats. A delta that names no change, written where the endpoint
  is valid, is no publication (END-3): it marks nothing modified, moves no
  `last_modified` (END-6), and notifies no subscriber. Further writes to
  the already modified endpoint in the same evaluation update its
  publication without notifying again. A downstream
  node runs at most once per cycle and reads the completed value and delta.
  Activity belongs to each input binding and controls whether it schedules
  its node. When the node runs, a publication on a passive input still
  satisfies `modified(input)` in that cycle. `is_active(input)` returns
  the current activity of the receiving node's direct input. A handler
  that requires both writes `when modified(input) && is_active(input)`.
  Active and passive inputs read the same validity, payload and delta.
- **END-6.** `last_modified(value)` returns `datetime?`. It returns null while
  the endpoint is invalid, including before its first publication and after
  invalidation. While valid, it returns the retained publication's UTC
  evaluation time. Silence preserves that time and an equal publication
  supplies its own time. A publication with a null payload has a timestamp.
- **END-7.** `valid(x)` of a list, a tuple, a struct or a dictionary returns true
  where any element, field or entry is valid, and for a published empty
  list or dictionary, which has none. `all_valid(x)` returns true
  where every one is valid by its own `valid`, one level deep, but for an
  optional child, whose type is `T?` or a field the language's TYPE-4 makes
  optional, which it does not ask: for a list of lists, each contained list
  is present and has a valid element or is a published empty list. It
  returns true for a valid empty collection. It takes only these
  collections and a window (END-9; the language's TIME-9).
  The [collection validity cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/collection_ticks.hgl)
  check a fixed list, a tuple and a struct, and a struct with an optional
  field. Proposed: `all_valid(x)` of a collection that is not valid
  returns false.
- **END-8.** A child of a collection is an endpoint whether or not it is
  valid: `valid`, `modified`, `delta_value` and `last_modified` take it,
  and `all_valid` where it is a collection, `valid(parts[1])`. Its value
  and its delta are read where the child is admitted (the language's
  TIME-1). A read at an index known only when it runs, `parts[i]`, of a
  child that is not valid fails with `value.unset_read`, or gives an `err`
  of an `error` with that code where its place expects a
  `result<T, error>` (the language's ERR-14). A read of a struct, a tuple
  or a fixed list whole is admitted where it is all valid, a child among
  them by its own `all_valid`, `all_valid(rows[0])`; in the value
  read, an optional child that is not valid is null, and its own `valid`
  tells that from a published null. One checked when it runs, `rows[i]`
  read whole, that is not all valid fails or gives an `err` as a child's
  read does. A dictionary or an unsized list is read whole where it is
  valid (the language's TIME-1). A position of an unsized list that holds
  no value reads in the whole value as the language's TIME-1 table has it
  for the element type: NaN for an `f64`, null for a `V?`, and for every
  other element type, which has no value for it, the read fails with
  `value.unset_read`, or gives an `err` where its place expects a
  `result<T, error>`.
  The
  [collection cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/collection_ticks.hgl)
  check both reads and a whole read, and the
  [rejection cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/rejections/collection_children.hgl)
  an unadmitted read at a constant index, of a field, and of a struct
  whole.
- **END-9.** A window (the language's TYPE-11) holds the values that arrived
  at it, oldest first. `valid(x)` of one returns true once it holds a
  value, and `all_valid(x)` where it holds at least its minimum: of a tick
  count, that many values; of a duration, values whose arrival times span
  at least the minimum, from the oldest held to the newest. A minimum of
  zero is held by the first value, and an empty window is not all valid.
  A window of a tick count drops its oldest value only at an arrival. In
  one of a duration, a value is still held when its age is the maximum,
  and leaves one smallest step later (CRUN-3), whether or not a value
  arrives then. The window ticks for that exit: it is modified, and
  notifies its active subscribers as a publication does (END-5). Each
  value leaves in a cycle of its own, and an exit and an arrival in one
  cycle are one tick. When the last value leaves, the window is empty. So
  `all_valid(x)` becomes true only at an arrival, and may become false at
  an exit. A read of a window's value gives a `list<V>` of the values
  held, oldest first. Before the
  window is all valid that read fails with `value.unset_read`, or gives an `err` of an `error`
  with that code where its place expects a `result<list<V>, error>` (the
  language's ERR-14). A lifted value function is given the window only
  where it is all valid: one that is valid and not all valid does not
  invoke it, though its parameter has a default (the language's FN-11).
  Proposed: a window emptied by exits stays valid; where a value leaves
  and none arrives, `delta_value(x)` has no value, and its read fails with
  `value.unset_read`; and `last_modified(x)` is then the exit's time.
  The
  [window cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/window_ticks.hgl)
  check a window of ticks, its read where a `result` is taken, and a
  lifted parameter with a default; and, in a window of a duration, an
  exit with nothing arriving, `all_valid` falling back, a minimum of zero,
  and the proposed validity of a window emptied by exits and `delta_value`
  at an exit.
- **END-10.** A reference, a `ref` parameter or result (the language's
  FN-16), is modified where its designation changes (the language's
  TIME-10). A publication of the designation it holds is no tick: unlike a
  value's (END-2, END-5), it leaves the endpoint unmodified and notifies no
  subscriber. A `ref` parameter of T given a connection of T designates
  that connection's endpoint. Returning a whole `ref` parameter samples
  its current designation. Its reader continues to receive publications
  from that target across later changes to the parameter's designation,
  until the node writes the result again. Returning an ordinary input or
  a child path such as `items[idx]` designates that logical endpoint,
  valid or not. Readers follow its binding (FN-16). A `ref` result designates one
  endpoint whole: an assignment to a part of its `out`, `out.bid = x` or
  `out[0] = x`, is rejected. So is `ref` on a `signal` parameter, which
  has no payload to designate (the language's FN-16, TYPE-12). Proposed:
  on a `ref` parameter,
  `modified` is whether its designation changed, not whether its target
  published; `valid` whether it designates something; and `last_modified`
  when its designation last changed. The
  [reference cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/reference_ticks.hgl)
  check repeated publication, sampling a `ref` parameter, and designating a
  list element. The
  [rejection cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/rejections/references.hgl)
  cover a read of a `ref` parameter's value and of its `delta_value`, an
  assignment to a part of a `ref` result's `out`, and a `signal`
  parameter marked `ref`.

## Publication and silence

`P` is the output of `publish_nullable` and `V` is the output of
`read_validity`. A probe tick lets `read_validity` inspect `P` at every cycle.

```hgl
module examples.scalar_validity

fn publish_nullable(command: i64) -> i64? {
    when {
        if command == 0 { return null }
        return command
    }
}

fn read_validity(probe: bool, watched: i64?) -> bool {
    when modified(probe) && valid([]) { return valid(watched) }
}

fn observe_validity(command: i64, probe: bool) -> bool =>
    read_validity(probe, publish_nullable(command))

test publication_establishes_validity {
    assert eval(observe_validity(
                command: [_, 0, _, 7],
                probe: [true, true, true, true])) == [false, true, true, true]
}
```

| Cycle | Command tick | Probe tick | `P` tick | `P` retained payload | `V` tick |
| --- | --- | --- | --- | --- | --- |
| `t1` | `_` | `true` | `_` | Unavailable | `false` |
| `t2` | `0` | `true` | `null` | `null` | `true` |
| `t3` | `_` | `true` | `_` | `null` | `true` |
| `t4` | `7` | `true` | `7` | `7` | `true` |

## Equal publications and idle cycles

`P` is the output of `publish_value` and `M` is the output of
`read_modified`. The probe ticks every cycle.

```hgl
module examples.scalar_modified

fn publish_value(value: i64) -> i64 {
    when { return value }
}

fn read_modified(watched: i64, probe: bool) -> bool {
    when modified(probe) && valid([]) { return modified(watched) }
}

fn observe_modified(value: i64, probe: bool) -> bool =>
    read_modified(publish_value(value), probe)

test equal_publications_are_modified {
    assert eval(observe_modified(
                value: [_, 7, _, 7],
                probe: [true, true, true, true])) == [false, true, false, true]
}
```

| Cycle | Value input tick | Probe tick | `P` tick | `P` retained value | `M` tick |
| --- | --- | --- | --- | --- | --- |
| `t1` | `_` | `true` | `_` | Unavailable | `false` |
| `t2` | `7` | `true` | `7` | `7` | `true` |
| `t3` | `_` | `true` | `_` | `7` | `false` |
| `t4` | `7` | `true` | `7` | `7` | `true` |

## Invalidation and recovery

`C` is the command input, `P` is the output of `produce`, and `W` is the
output of `watch`. The selector `valid([])` allows `watch` to report an
invalid input.

```hgl
module examples.scalar_invalidation

fn produce(command: i64) -> i64 {
    inject out
    when {
        if command != 0 { return command }
        invalidate(out)
    }
}

fn watch(input: i64) -> bool {
    when valid([]) { return valid(input) }
}

fn observe(command: i64) -> bool => watch(produce(command))
```

| Cycle | `C` tick | Action on `P` | `valid(P)` | `modified(P)` | `delta_value(P)` | `last_modified(P)` | `W` tick |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `t1` | `7` | Publish `7` | true | true | `7` | `t1` | `true` |
| `t2` | `0` | Invalidate | false | false | Unavailable | `null` | `false` |
| `t3` | `0` | Invalidate again | false | false | Unavailable | `null` | `_` |
| `t4` | `7` | Publish `7` | true | true | `7` | `t4` | `true` |

The `false` tick shows that invalidation wakes `watch`. The `_` shows
silence on repeated invalidation. The
[core eval case](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/scalar_endpoints.hgl)
checks the `W` trace.

## Multiple writes in one cycle

This illustration of END-5 uses proposed set delta syntax and semantics.
`C` is the command input, `S` is the output of `change_members`, and `R` is
the output of `read_delta`.

```hgl
module examples.set_updates

fn change_members(command: i64) -> set<i64> {
    inject out
    when {
        if command == 1 { return delta<set<i64>>(added: [1]) }
        out = delta<set<i64>>(added: [2, 3])
        out = delta<set<i64>>(removed: [1, 2])
    }
}

fn read_delta(input: set<i64>) -> set<i64> {
    when { return delta_value(input) }
}

fn observe_set_updates(command: i64) -> set<i64> =>
    read_delta(change_members(command))

test set_writes_coalesce {
    assert eval(observe_set_updates(command: [1, 2])) == [
        delta<set<i64>>(added: [1]),
        delta<set<i64>>(added: [3], removed: [1])
    ]
}
```

| Cycle | `C` tick | Writes to `S` by `change_members` | `S` at cycle end | `read_delta` evaluations | `R` tick |
| --- | --- | --- | --- | --- | --- |
| `t1` | `1` | Add `1` | `{1}` | 1 | Add `1` |
| `t2` | `2` | Add `2, 3`, then remove `1, 2` | `{3}` | 1 | Add `3`, remove `1` |

At `t2`, the addition and removal of `2` cancel in the set delta. The
[core eval fixture](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/set_endpoints.hgl)
checks the published `R` trace.

## Proposed examples and conformance observations

Bounded scalar-output trace at increasing evaluation times:

| Event | valid | modified | Retained payload | last_modified |
| --- | --- | --- | --- | --- |
| Initial invalid state | false | false | Unreadable | null |
| Publish 7 at t1 | true | true | 7 | t1 |
| Idle cycle t2 | true | false | 7 | t1 |
| Publish 7 again at t3 | true | true | 7 | t3 |
| Invalidate at t4 | false | false | Unreadable | null; active subscribers notified |
| Publish null on an i64? endpoint at t5 | true | true | Present null | t5 |

The last row demonstrates the nullable scalar case, not a type change to an
existing i64 endpoint. A payload guard and a metadata-presence guard observe
different facts. False, zero, and empty text need the same presence controls.

Either input can wake this function. It forwards publications from `a`.

```hgl
module examples.scalar_delta

fn forward_a(a: i64, b: i64) -> i64 {
    when {
        if valid(a) && modified(a) {
            return delta_value(a)
        }
    }
}
```

With both inputs valid, a tick from `b` alone leaves the output silent.
Removing the guard makes the delta read a source error.
