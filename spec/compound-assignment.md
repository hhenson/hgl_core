# Compound assignment

Status: ASG-1, ASG-2 and ASG-6 are agreed. ASG-3, ASG-4, ASG-5 and ASG-7,
the identities' names and every declaration below are proposed. `hgl/`
declares none of them yet.

## Scope and API

The [language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/expressions-and-bindings.md)
says that a compound assignment binds an identity whose first parameter is
a `var`, that in composition it is a rebinding, and that a place in `out`
is given as its endpoint (EXPR-18). This contract has the identities and
core's candidates.

| Syntax | Identity | Default |
| --- | --- | --- |
| `a += b` | `iadd_(a, b)` | `a = a + b` |
| `a -= b` | `isub_(a, b)` | `a = a - b` |
| `a *= b` | `imul_(a, b)` | `a = a * b` |
| `a /= b` | `idiv_(a, b)` | `a = a / b` |
| `a //= b` | `ifloordiv_(a, b)` | `a = a // b` |
| `a %= b` | `imod_(a, b)` | `a = a % b` |

Reference declarations, `iadd_` standing for all six:

```hgl
export operator iadd_<L, R>(lhs: L, rhs: R)

# ASG-3: what the symbol gives, assigned.
impl const fn iadd_<T, R>(var lhs: T, rhs: R)
    requires add_(T, R) -> T {
    lhs = lhs + rhs
}

# ASG-5: a place in `out`.
native impl const fn iadd_<T, R>(target: endpoint<T>, rhs: R)
    requires add_(T, R) -> T

# TYPE-1: text grows in place.
native impl const fn iadd_(var lhs: str, rhs: str)

# ASG-4: one more entry.
impl const fn iadd_<C, T>(var lhs: C, rhs: T)
    requires C in {list<T>, set<T>} {
    append(lhs, rhs)
}

export operator append<C, T>(var collection: C, item: T)
    requires C in {list<T>, set<T>}
```

## Rules

- **ASG-1.** Each compound assignment binds its own identity, and every
  value candidate of one takes its first parameter as a `var` (the
  language's FN-15): the call assigns the place, and gives no result.
- **ASG-2.** A candidate may declare options, given in braces after the
  token (the language's EXPR-10): `count //={zero: .fail} step`.
- **ASG-3.** Proposed: core's default candidate of each identity is the
  symbol's result assigned, for two types whose symbol gives the place's
  type. So a type that declares `add_(T, R) -> T` has `+=`, and a more
  specific candidate is selected before the default (the language's GEN-5).
  Where the symbol gives another type there is no candidate: `count /= 2`
  for an `i64` rejects (`operator.no_candidate`), `/` giving an `f64`.
- **ASG-4.** Proposed: `+=` on a list or a set is `append`: one entry, the
  right operand. `append<C, T>(var collection: C, item: T)` is an operator
  with a candidate for a `list<T>` ([list operations](list-operations.md))
  and one for a `set<T>`, which leaves a set that holds the member as it
  was. The one shape leaves nothing to choose between: `rows += []` for a
  `list<list<i64>>` appends an empty row. A list is not appended to a list
  of its own type by `+=`, which rejects; `extend` does that.
- **ASG-5.** Proposed: for a place in `out`, core's candidate takes
  `endpoint<T>` and publishes what the default would assign, as an
  assignment to that place does (the language's TIME-7).
- **ASG-6.** `text += piece` appends in place, at the cost of the piece
  (the language's TYPE-1).
- **ASG-7.** Proposed: `/=`, `//=` and `%=` take `zero` as their symbol
  does (NUM-12), with the policies whose result is the place's type. A
  policy that gives a `result`, `.error`, has no candidate: the statement
  has nowhere to put it.

## Examples

```hgl
const fn tally(words: list<str>) -> str {
    var count = 0
    var seen: list<str> = []
    var text = ""
    for word in words {
        count += 1
        seen += word
        text += word
    }
    return text
}
```

Deliberate source rejections: `count /= 2` for `var count: i64`
(`operator.no_candidate`); `seen += seen` for a `list<str>`, a list given
where an entry is taken (`operator.no_candidate`); `fixed += 1` for
`let fixed = 0` (`binding.immutable`).
