# Scalar temporal endpoint observations

Status: END-1 through END-11 are agreed. `hgl/` still declares
`invalidate` with a plain endpoint.

## Scope and API

This slice covers owned scalar outputs and fixed, directly bound scalar
inputs during evaluation. Nullable scalar values are included. END-7 gives
the validity of a collection, END-8 the read of its child, END-9 a window's
readiness and what it holds, END-10 when a reference ticks, and END-11
which operation writes its endpoint. Other collection
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
| `delta_value(value)` | The current cycle's `delta<T>` (TYPE-16), with the read form chosen by its place. A duration window takes the result and nullable forms. See END-3 and END-9. |
| `removed_value(window)` | `result<V, error>`: the value that left the window in the cycle, or an `err` where none left; see END-9 |
| `is_active(input)` | bool; current activity of this direct input, including while invalid |
| `last_modified(value)` | Agreed `datetime?`; see END-6 |

`valid`, `all_valid`, `modified`, `delta_value`, `removed_value`, `last_modified`,
`invalidate`, `is_active`, `activate`, and `passivate` are declared in `hgl/native/endpoints.hgl` as
native value functions that take the endpoint and not its payload, an
`endpoint<T>` (the language's TIME-9), which `invalidate` alone takes as a
`var` (END-11). Runtime metadata reads create no graph
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
  For a window it is the value that arrived (END-9). For a list, the
  delta read gives the new size where it moves, never a negative one, and
  on a shrink names the dropped tail positions in `removed`, also where
  the delta written gave only a smaller `size_modified`, or a negative
  one, which reduces the list by that
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
  changes validity (the language's TIME-7). For a never-published or
  wholly invalidated `list<V>`, dictionary or set (END-4), it makes the
  endpoint valid, empty and modified, and `delta_value` returns that delta.
  A growing list with positions but no valid child keeps its size and
  stays invalid on a delta that names no change. It has no delta to read
  and wakes no subscriber. Writing `size_modified: 0` or publishing `[]`
  whole then publishes a valid empty list with the dropped positions in
  `removed`. Where the endpoint is already valid a delta that names no
  change is no publication (END-5). Such a delta of a struct, a tuple or a
  `list<V, N>` sets no child and changes no validity (END-7): it is never
  a tick. A delta whose every entry is a removal ignored as above names no
  change too, and ticks only where it makes the endpoint valid. The
  [collection cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/collection_ticks.hgl)
  check a list and a set, and the
  [dictionary cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/dictionary_ticks.hgl)
  a dictionary.
  An endpoint that does not tick in the cycle has no delta. An
  admitted `delta_value(x)` takes its form from its place (ERR-14, ERR-15).
  A `delta<T>` place gives the delta or fails with `value.unset_read`.
  A `result<delta<T>, error>` place gives `ok` of the delta or an
  `err` with that code. A `delta<T>?` place gives the delta or null.
  A window of a duration takes the result and nullable forms (TYPE-11, END-9).
  It alone can tick without a delta, so a modified endpoint of any
  other type gives `ok` in the result form. Delta reads follow the
  admission rules for the endpoint (TIME-1, TIME-4).
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
  `last_modified` (END-6), and notifies no subscriber. Nor is a collection
  given whole that changes nothing (the language's TIME-7): a set given
  the members it holds, or an empty list or dictionary given to a valid
  empty one. A dictionary's entries and a list's positions given again
  are publications, as an equal scalar is. Further writes to
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
  returns true for a valid empty collection. `all_valid(x)` of a
  collection that is not valid returns false. It takes only these
  collections and a window (END-9; the language's TIME-9).
  The [collection validity cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/collection_ticks.hgl)
  check a fixed list, a tuple and a struct, and a struct with an optional
  field.
- **END-8.** A collection's child is an endpoint whether or not it is
  valid. An untyped binding, `let entry = parts[i]`, holds the endpoint
  and reads no value. `valid(entry)` and other endpoint operations can
  inspect it. A use that requires a value reads it at that use, under
  the admission rules of TIME-1. For a child whose read is checked at
  runtime, a `T` place gives its value or fails with `value.unset_read`.
  A `result<T, error>` place gives `ok` of the value or an `err` with
  that code. A `T?` place gives the value or null (ERR-14, ERR-15).
  A struct, tuple or fixed list read whole needs its own `all_valid`,
  including when it is a child such as `rows[i]`. A checked whole read
  of one that is not all valid takes the same three forms. An unset optional
  part of an otherwise readable whole is null, and its own `valid`
  tells that case from a published null.
  A dictionary or unsized list is admitted for a whole read where it
  is valid. The dictionary omits wholly unset entries. An entry that
  is valid but not all valid makes the whole dictionary read take the
  same failure forms as an incomplete child. In a whole unsized list
  read, a gap gives NaN for an `f64` and null for a `V?`. For a gap
  whose element type has no value for it, the whole list read takes the
  same failure forms (TIME-1).
  The [endpoint read cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/endpoint_reads.hgl)
  check the read forms and the endpoint binding. The
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
  held, oldest first. Below `all_valid`, a `list<V>` place fails with
  `value.unset_read`, a `result<list<V>, error>` place gives an `err`
  with that code and a `list<V>?` place gives null (ERR-14). A lifted value function is given the window only
  where it is all valid: one that is valid and not all valid does not
  invoke it, though its parameter has a default (the language's FN-11).
  `delta_value(x)` of a window is the value that arrived in the cycle
  (the language's TYPE-16), read before the window is all valid as after.
  In a cycle in which a value only leaves a window of a duration, the
  window ticks and has no delta. In that cycle `delta_value(x)` gives an
  `err` coded `value.unset_read` in a `result<delta<T>, error>` place
  and null in a `delta<T>?` place (END-3). A place for `delta<T>`
  alone rejects a duration window's delta with `payload.unguarded` (TYPE-11).
  A window of a tick count ticks only at an arrival and takes the
  three forms of END-3.
  `removed_value(x)` is the value that left the window in the cycle: in
  one of a tick count, the oldest value an arrival pushes out; in one of a
  duration, the value that leaves by age, a value arriving then or not.
  It is read only where its place expects a `result<V, error>`, and gives
  `ok` of that value, or an `err` coded `value.unset_read` where none left
  (the language's ERR-14). A `V` or `V?` place rejects the read with
  `payload.unguarded` for every window (TYPE-11). A node publishes a V to
  a window, which arrives, and nothing else.
  The delta of a window of a nullable V, `rolling_window<f64?, 3>`, is
  the nullable's own delta, `delta<f64?>`, which holds what arrived, a
  null among them, so that a null that arrives is told from no change
  where the delta is a struct's field, a tuple's position or an entry
  (the language's TYPE-11, TYPE-16): `delta_value(x)` gives
  `delta<f64?>(value: 1.0)` for 1.0 and `delta<f64?>()` for a null. Such
  a window also takes its delta as the arrival it holds, as `replay`
  publishes a recorded one (the language's TIME-7). `removed_value(x)`
  of it gives the V itself, null included.
  A window emptied by exits stays valid. Where a value leaves and none
  arrives, `last_modified(x)` is the exit's time.
  The
  [window cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/window_ticks.hgl)
  check a window of ticks, its read where a `result` is taken, a lifted
  parameter with a default, and its delta and `removed_value` once it is
  full; in a window of a duration, an exit with nothing arriving,
  `all_valid` falling back, a minimum of zero, the result and nullable
  delta forms at an exit alone and at an exit with an arrival,
  `removed_value` at those ticks, and the
  validity of a window emptied by exits; and a recording of a window of a
  duration, which holds its arrivals, and its replay. The
  [window rejection cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/rejections/window_reads.hgl)
  cover a plain read of the delta of a window of a duration and of
  `removed_value`.
- **END-10.** A reference, a `ref` parameter or result (the language's
  FN-16), is a temporal value of a designation (the language's TIME-10).
  A runtime is held to what the use of a reference produces, the ticks
  that reach a reader and what wakes a node, and not to how it carries a
  reference between them. A reader of a `ref` result is an ordinary input
  given that result, one that is no `ref` parameter. It observes the
  endpoint designated as if wired to it: valid where that endpoint is,
  ticking where it publishes.
  Where the result comes to designate another
  endpoint, the reader ticks in that cycle with the value that endpoint
  holds, whether or not the endpoint publishes then; where the endpoint
  holds none, the reader is not valid, and is silent, until it publishes;
  and the endpoint left behind no longer reaches it. A result given the
  designation it holds gives its reader no tick: the reader goes on
  observing the endpoint, a publication of it in that cycle included. In
  a node, a `ref` parameter is passive to what it designates: a
  publication of that endpoint does not wake the node, and `valid` of
  the parameter is whether a reference is present, whatever the endpoint
  it designates holds. Where a `ref` parameter is active, the node is
  woken where the parameter is given a designation. A `ref`
  parameter of T given a connection of T designates
  that connection's endpoint. Returning a whole `ref` parameter samples
  its current designation. Its reader continues to receive publications
  from that target across later changes to the parameter's designation,
  until the node writes the result again. Returning an ordinary input or
  a child path such as `items[idx]` designates that logical endpoint,
  valid or not. Readers follow its binding (FN-16). A `ref` result designates one
  endpoint whole: an assignment to a part of its `out`, `out.bid = x` or
  `out[0] = x`, is rejected. So is `ref` on a `signal` parameter, which
  has no payload to designate (the language's FN-16, TYPE-12), and so is
  `ref` on a parameter or a result whose type is a window. The code of
  that rejection is `ref.position`, and `removed_value` is not read of a
  window that is a child of a `ref` parameter or of `out` of a
  `ref` result, as `delta_value` is not (the language's FN-16). The
  [reference cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/reference_ticks.hgl)
  check what a reader observes of a designation given again, of a change
  to a target that holds a value and to one that holds none, and of a
  list element; a relay that samples its `ref` parameter; `valid` of a
  `ref` parameter whose target holds no value; a reader's validity
  while its target holds no value; a node
  woken by its `ref` parameter and not by its target; and a relay that
  follows its `ref` parameter. The
  [rejection cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/rejections/references.hgl)
  cover a read of a `ref` parameter's value and of its `delta_value`, an
  assignment to a part of a `ref` result's `out`, a `signal` parameter
  marked `ref`, and a window parameter marked `ref`.
- **END-11.** `invalidate` writes the endpoint it is given, so it takes
  it as a `var` (the language's FN-15): `out`, whole, and nothing else. It
  is whole-output invalidation (the language's TIME-7): a place in `out`,
  `invalidate(out.bid)` or `invalidate(out[0])`, rejects, and no rule
  gives a part of an output an invalidation of its own. A temporal input
  rejects, being read-only (`binding.immutable`), and so does a call
  outside a `when`, as any write of `out` does (`out.lifecycle`). Every
  other operation here takes a plain `endpoint<T>`. `activate` and
  `passivate` act on the node's subscription to an input and leave the
  input as it is.

  Accepted fragment, in a node `fn gate(price: f64, open: bool) -> f64`
  that injects `out`; conformance planned:

  ```hgl
  when valid(price, open) {
      if open { return price }
      invalidate(out)               # the output, whole
      passivate(price)              # an action on the subscription
  }
  ```

  Deliberate source rejections there: `invalidate(price)`
  (`binding.immutable`); `invalidate(out)` in `stop` (`out.lifecycle`).

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
    assert eval observe_validity(
                command: [_, 0, _, 7],
                probe: [true, true, true, true]) == [false, true, true, true]
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
    assert eval observe_modified(
                value: [_, 7, _, 7],
                probe: [true, true, true, true]) == [false, true, false, true]
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
    assert eval observe_set_updates(command: [1, 2]) == [
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
