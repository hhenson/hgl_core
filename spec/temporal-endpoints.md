# Scalar temporal endpoint observations

Status: END-1 and END-3 through END-6 are agreed. END-2 is proposed.

## Scope and API

This slice covers owned scalar outputs and fixed, directly bound scalar
inputs during evaluation. Nullable scalar values are included. Collections,
references, sampled/rebound inputs, and source scheduling retain their separate
contracts. The [language temporal model](https://github.com/hhenson/hgl_spec/blob/main/spec/language/temporal-model.md)
owns endpoint interpretation, handler selection, and phase admission.

| HGL observation | Runtime result / admission |
| --- | --- |
| `valid(value)` | bool; may inspect an invalid endpoint |
| `modified(value)` | bool; whether the endpoint published in the current cycle |
| Runtime parameter payload | T; requires readable valid input, subject to declared nullable/default argument binding |
| `delta_value(value)` | Agreed `T` for scalar inputs. See END-3. |
| `last_modified(value)` | Agreed `datetime?`; see END-6 |

These are semantic interfaces, not native declarations. Runtime metadata reads
create no graph nodes. Composition forms and empty/multi-argument handler
selectors follow the language contract rather than this one-endpoint table.

## Proposed rules

- **END-2.** A publication marks the owned endpoint modified at the current
  evaluation time, including when its value equals the previous value. In a
  later idle cycle it is unmodified while its value remains retained. A fixed
  directly bound input observes its output's state.

## Agreed rules

- **END-1.** A scalar publication makes the endpoint valid and retains its
  payload, including false, zero, empty text, null, NaN, or infinity.
  Silence preserves validity and the retained payload.
- **END-3.** `delta_value(x)` returns the scalar value `x` published this cycle,
  including a published null. The checker must prove `x` is valid and modified
  at the call.
- **END-4.** Invalidation clears validity, modification, payload, and delta.
  The transition to invalid notifies active subscribers. Each handler's
  validity requirements still apply. Repeated invalidation while invalid is
  silent. Within a producer evaluation the last action determines the state:
  publish then invalidate leaves the endpoint invalid. Invalidate then publish
  leaves it valid and modified.
- **END-5.** A publication notifies active subscribers, including when its
  value repeats. Further writes to the already modified endpoint in the same
  evaluation update its publication without notifying again. A downstream
  node runs at most once per cycle and reads the completed value and delta.
  Active and passive inputs read the same endpoint state.
- **END-6.** `last_modified(value)` returns `datetime?`. It returns null while
  the endpoint is invalid, including before its first publication and after
  invalidation. While valid, it returns the retained publication's UTC
  evaluation time. Silence preserves that time and an equal publication
  supplies its own time. A publication with a null payload has a timestamp.

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

fn read_validity(watched: i64?, probe: bool) -> bool {
    when modified(probe) && valid() { return valid(watched) }
}

fn observe_validity(command: i64, probe: bool) -> bool =>
    read_validity(publish_nullable(command), probe)

test publication_establishes_validity {
    assert eval(observe_validity,
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

## Invalidation and recovery

`C` is the command input, `P` is the output of `produce`, and `W` is the
output of `watch`. The empty validity selector allows `watch` to report an
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
    when valid() { return valid(input) }
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
    assert eval(observe_set_updates, command: [1, 2, _]) == [
        delta<set<i64>>(added: [1]),
        delta<set<i64>>(added: [3], removed: [1]),
        _
    ]
}
```

| Cycle | `C` tick | Writes to `S` by `change_members` | `S` at cycle end | `read_delta` evaluations | `R` tick |
| --- | --- | --- | --- | --- | --- |
| `t1` | `1` | Add `1` | `{1}` | 1 | Add `1` |
| `t2` | `2` | Add `2, 3`, then remove `1, 2` | `{3}` | 1 | Add `3`, remove `1` |
| `t3` | `_` | None | `{3}` | 0 | `_` |

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
