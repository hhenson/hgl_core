# Scalar operators and conversion

Status: agreed operator domains for the scalar types other than numbers, the
comparison outcome type, and the domains of the conversion operator. What
each conversion yields is not specified here. No executable conformance or
native ABI is specified.

## Contract

The [numeric contract](numeric-operators.md) covers `i64` and `f64`. This
chapter covers the other scalar types. The
[language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/expressions-and-bindings.md)
owns symbol syntax, precedence, and phase checking. An instant is a
`datetime`, `date`, `civil_datetime`, or `zoned_datetime`.

- **OPS-1.** The symbol operators admit the operand types of the first table
  below, beside those of the numeric contract. Every other pairing is a
  source rejection, `operator.no_candidate`. An operation that commutes admits
  its operands in either order; one that does not admits the order shown.
- **OPS-2.** `<`, `<=`, `>`, `>=`, and `<=>` compare two values of one type
  among `str`, `bool`, `date`, `time`, `datetime`, `civil_datetime`, and
  `duration`. `==` and `!=` compare two values of one type. The zoned types
  have no order: same-instant zoned values need not be equal. Numbers
  compare as the numeric contract says, an `i64` with an `f64` included.
- **OPS-3.** `CmpResult` is the enum of comparison outcomes, with the members
  `less`, `equivalent`, `greater`, and `unordered`. `<=>` returns one
  (NUM-6). Only a numeric comparison yields `unordered`. The other
  comparison forms return a bool.
- **OPS-4.** `convert<T>(value)` converts a value to the type T. An integer
  literal is an `f64` where one is expected; no other value changes type
  without it. It admits the conversions of the second table below and
  rejects every other.
- **OPS-5.** `&&`, `||`, and `!` take bools. A number or a string has no
  truth value.

| Operator | Operands | Result |
| --- | --- | --- |
| `+` | `str` and `str` | `str` |
| `+` | `duration` and `duration` | `duration` |
| `+` | An instant and a `duration`, in either order | The instant's type |
| `+` | `date` and `time` | `civil_datetime` |
| `-` | `duration` less `duration` | `duration` |
| `-` | An instant less a `duration` | The instant's type |
| `-` | `datetime` less `datetime`, `date` less `date`, `civil_datetime` less `civil_datetime` | `duration` |
| `*` | `duration` and `i64` or `f64`, in either order | `duration` |
| `*` | `str` and `i64`, in either order | `str` |
| `/` | `duration` by `i64` or `f64` | `duration` |
| `/` | `duration` by `duration` | `f64` |
| `//` | `duration` by `duration` | `i64` |
| `-x` | `duration` | `duration` |
| `<`, `<=`, `>`, `>=` | Two of one ordered type (OPS-2) | `bool` |
| `<=>` | Two of one ordered type (OPS-2) | `CmpResult` |
| `==`, `!=` | Two values of one type | `bool` |

| `convert` from | To |
| --- | --- |
| `i64` | `f64`, `bool`, `str` |
| `f64` | `i64`, `bool`, `str` |
| `bool` | `i64`, `f64`, `str` |
| `str` | `i64`, `f64`, `bool`, `bytes` |
| `bytes` | `str` |
| `date` | `datetime` |
| `datetime` | `date` |

## Examples

Accepted expression fragments, with `count` an `i64` in scope:

```hgl
"a" + "b"
5m + 30s
@2026-09-03T09:30Z + 5m
5m + @2026-09-03T09:30Z
@2026-09-04 - @2026-09-03
2 * 5m
5m / 2m
5m <=> 30s
convert<f64>(count)
```

Their types are `str`, `duration`, `datetime`, `datetime`, `duration`,
`duration`, `f64`, `CmpResult`, and `f64`. Deliberate source rejections under
OPS-1: `5m - @2026-09-03T09:30Z`, a duration less an instant; `"a" + 1`;
`@2026-09-03 + @2026-09-04`. Under OPS-5: `1 && true`. Executable conformance
is planned.
