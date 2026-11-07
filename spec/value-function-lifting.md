# Value-function lifting observations

Status: agreed admission, binding, aggregate, and ownership observations.
Executable conformance is planned; this document supplies no native API.

The [language rules FN-6 and FN-11 through FN-13](https://github.com/hhenson/hgl_spec/blob/main/spec/language/functions-and-phases.md)
own source interpretation. Core owns the following runtime expectations.
Ordinary validity comes from each input type's contract; lifting adds no
recursive full-validity or fixed-size condition to an unsized list.

## Defaulted temporal argument

Accepted value-function fragment:

```hgl
const fn scale(value: f64, factor: f64 = 2.0) -> f64 => value * factor
```

In composition, supply temporal inputs for both parameters. Initially both
are invalid; a dash below means no publication, not invalidation. Valid inputs
retain their values across silence.

| Cycle | Value publication | Factor publication | Expected output |
| --- | --- | --- | --- |
| 0 | 1.0 | — | 2.0, using the default factor |
| 1 | — | 4.0 | 4.0, using retained value 1.0 |
| 2 | 3.0 | — | 12.0, using retained factor 4.0 |
| 3 | 3.0 | — | 12.0, including the repeated publication |

## Aggregate argument

Accepted value-function fragment:

```hgl
const fn total(values: list<f64>) -> f64 {
    var result: f64 = 0.0
    for value in values {
        result += value
    }
    return result
}
```

For a temporal list input, invoke `total` with the current ordinary list when
`valid(list_value)` holds. A valid `[1.0, 2.0, 3.0]` yields 6.0. A later
second-element change giving a valid `[1.0, 5.0, 3.0]` yields 9.0. The helper
is invoked once with the list; no per-element lifting is introduced.

## Further conformance observations

| Condition | Expected observation |
| --- | --- |
| Required non-nullable input without a default is invalid | No helper invocation or output publication |
| Nullable input without a default is invalid, other required inputs valid | Invoke with null |
| Defaulted input is invalid, other required inputs valid | Invoke with the declared default |
| An input admitting null has a default and publishes a valid null | Invoke with null, without selecting the default |
| No input notification to an existing lifted node, but a fallback is available | Fallback alone creates no evaluation or tick |
| Valid unsized list changes length | Use the current list; no required N is inferred |
| Valid empty unsized list passed to `total` | Invoke and publish 0.0 |
| Fixed-size list or window has not met its type's readiness contract | Preserve that contract; lifting does not admit unreadable required data |
| Temporal input changes after a returned value is retained | Earlier retained result remains unchanged |
| Helper uses an admitted borrowed view | View stays within its permitted lifetime; lifting does not grant ownership |

## Explicit nullable values

Accepted value-function fragment using the language's `T?` annotation:

```hgl
const fn nullable_probe(value: f64? = 2.0, trigger: i64) -> f64? => value
```

Supply temporal arguments for both parameters. `trigger` provides the indicated
notification while `value` exercises nullable/default binding:

| Value input at a trigger publication | Expected published result |
| --- | --- |
| Invalid | 2.0 from the declared default |
| Valid null | A present null value |
| Valid 3.0 | 3.0 |

With the same nullable parameter and no default, invalid value input binds and
publishes null on an admitted trigger publication. A valid null publication is
data, not silence or invalidity. A guarded helper returning f64 instead may
map it to a non-null value, as in the language's `value_or_zero` illustration.

The language distinguishes `list<f64?>` from `list<f64>?`; type syntax does not
add a recursive `all_valid` condition to ordinary unsized-list admission.
Test packaging, replay, and recordings retain their own status. Executable
conformance remains planned.

## Aggregate validity and retention controls

For the language's `count_items(values: list<f64>) -> i64` value helper, use a
fixture exposing a valid unsized list of length 2 with one invalid child.
`all_valid` is false. One invocation reads only length and publishes 2. When
that child becomes valid, another invocation publishes 2 again; adding a valid
third child later publishes 3. No invalid child payload is read, no fixed size
is inferred, and no elementwise calls replace the whole-list invocation.

For the language's `keep_value(values: list<i64>) -> list<i64>` helper,
retain R0 from complete valid input `[1, 2]`, then publish `[1, 7]` and retain
R1. R0 remains `[1, 2]`; R1 is `[1, 7]`. The retained results must not alias
subsequently changing producer storage.

A provider-declared view limited to its supplying invocation cannot itself be
retained beyond that invocation. Source/provider lifetime checking must reject
such an escape; retaining a permitted independent owning value is accepted.
These controls introduce no public native/view API. Executable conformance is
planned.
