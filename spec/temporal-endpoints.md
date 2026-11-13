# Scalar temporal endpoint observations

Status: scalar delta (END-3) and timestamp (END-6) are agreed. The other core
rules are proposed.

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

- **END-1.** A scalar publication establishes validity and a retained value.
  Until invalidation, later silence preserves both. Validity is independent
  of whether a value is false, zero, empty text, null, NaN, or infinity.
- **END-2.** A publication marks the owned endpoint modified at the current
  evaluation time, including when its value equals the previous value. In a
  later idle cycle it is unmodified while its value remains retained. A fixed
  directly bound input observes its output's state.
- **END-4.** Invalidation clears validity. The endpoint is unmodified and has
  no readable ordinary payload or scalar delta. Invalidation still notifies
  active subscribers and can cause scheduling without a modified input.
  Required validity and nullable/default bindings determine admission.
  Invalidating an already invalid endpoint changes nothing and notifies no
  subscriber. Within one producer evaluation the last action decides the
  final state: publish then invalidate ends invalid; invalidate then publish
  ends valid and modified.
- **END-5.** Active/passive status affects scheduling, not the observations
  above. An equal publication still notifies. Repeated writes while already
  modified in one producer evaluation expose the final value/delta downstream
  without additional publication notifications. Normal graph scheduling still
  evaluates each node at most once per cycle.

## Agreed rules

- **END-3.** `delta_value(x)` returns the scalar value `x` published this cycle,
  including a published null. The checker must prove `x` is valid and modified
  at the call.
- **END-6.** `last_modified(value)` returns `datetime?`. It returns null while
  the endpoint is invalid, including before its first publication and after
  invalidation. While valid, it returns the retained publication's UTC
  evaluation time. Silence preserves that time and an equal publication
  supplies its own time. A publication with a null payload has a timestamp.

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
