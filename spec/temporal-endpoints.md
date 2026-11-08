# Scalar temporal endpoint observations

Status: agreed scalar timestamp rule (END-6); the rest of the core slice is
proposed. The language's publication, phase, and guarded read rules retain
their agreed status.

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
| `delta_value(value)` | T for this scalar slice; requires that particular endpoint valid and modified |
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
- **END-3.** Scalar delta access returns the current publication's value,
  including a present null for T?. It does not substitute a retained value for
  an unmodified endpoint. The source must prove valid and modified for the
  particular endpoint; an any-input notification alone is insufficient.
- **END-4.** Invalidation clears validity. The endpoint is unmodified and has
  no readable ordinary payload or scalar delta. Invalidation still notifies
  active subscribers and can cause scheduling without a modified input.
  Required validity and nullable/default bindings determine admission.
- **END-5.** Active/passive status affects scheduling, not the observations
  above. An equal publication still notifies. Repeated writes while already
  modified in one producer evaluation expose the final value/delta downstream
  without additional publication notifications. Normal graph scheduling still
  evaluates each node at most once per cycle.

## Agreed timestamp rule

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

Proposed compute fixture using the agreed guarded-delta source form:

```hgl
fn scalar_copy(value: i64) -> i64 {
    when valid(value) && modified(value) {
        return delta_value(value)
    }
}
```

Its publications preserve equal input ticks. A required invalid input prevents
the handler's payload read. Deliberate source rejection: in a two-input handler,
reading one endpoint's scalar delta with only an any-input activation guarantee
and no proof that this endpoint is valid and modified. Core owns executable
cases and expected observations; implementation and conformance remain planned.
