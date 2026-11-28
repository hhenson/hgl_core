# Compound assignment

Status: ASG-1 through ASG-7 and the identities' names are agreed. The
declarations below are reference declarations. `hgl/` declares none of
them yet, and declares `append` and `extend` as native functions of a list.

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
export operator iadd_<L, R>(var lhs: L, rhs: R)

# ASG-3: what the symbol gives, assigned.
impl const fn iadd_<T, R>(var lhs: T, rhs: R)
    requires add_(T, R) -> T {
    lhs = lhs + rhs
}

# ASG-5: a place in `out`.
native impl const fn iadd_<T, R>(var target: endpoint<T>, rhs: R)
    requires add_(T, R) -> T

# TYPE-1: text grows in place.
native impl const fn iadd_(var lhs: str, rhs: str)

# ASG-4: an entry is appended, a collection of the place's type extends it.
impl const fn iadd_<C, T>(var lhs: C, rhs: T)
    requires C in {list<T>, set<T>} { append(lhs, rhs) }
impl const fn iadd_<C, T>(var lhs: C, rhs: C)
    requires C in {list<T>, set<T>} { extend(lhs, rhs) }

export operator append<C, T>(var collection: C, item: T)
    requires C in {list<T>, set<T>}
export operator extend<C, T>(var collection: C, items: C)
    requires C in {list<T>, set<T>}
```

## Rules

- **ASG-1.** Each compound assignment binds its own identity, whose
  declaration marks its first parameter `var` (the language's FN-15): every
  candidate takes a place, or an output, there. The call assigns it and
  gives no result.
- **ASG-2.** A candidate may declare options, given in braces after the
  token (the language's EXPR-10): `count //={zero: .fail} step`.
- **ASG-3.** Core's default candidate of each identity is the symbol's
  result assigned, `a = a op b`, for two types whose symbol gives the
  place's type. It is selected where no more specific candidate of the
  identity is declared (the language's GEN-5). So a type that declares
  `add_(T, R) -> T` has `+=`. Where the symbol gives another type there
  is no candidate: `count /= 2` for an `i64` rejects
  (`operator.no_candidate`), `/` giving an `f64`.
- **ASG-4.** `+=` on a list or a set has two candidates. Given an entry,
  it is `append`. Given a collection of the place's own type, it is
  `extend`. `append` and `extend` are operators, each with a candidate
  for a list ([list operations](list-operations.md)) and one for a set:
  `append` leaves a set that holds the member as it was, and `extend`
  puts each member the set has not. Where the right operand's type fits
  both, the empty literal given to a collection of collections,
  `rows += []` for a `list<list<i64>>`, the statement is ambiguous
  (`operator.ambiguous`), and `append(rows, [])` or `extend(rows, [])`
  says which.
- **ASG-5.** For a place in `out`, core's candidate takes a `var`
  `endpoint<T>`, an output, and publishes what the default would assign, as an
  assignment to that place does (the language's TIME-7).
- **ASG-6.** `text += piece` appends in place, at the cost of the piece
  (the language's TYPE-1).
- **ASG-7.** `/=`, `//=` and `%=` take `zero` as their symbol
  does (NUM-12), and the default (ASG-3) admits each policy whose result
  is the place's type. So `.error`, which gives a `result`, is taken by
  a place that holds one: for `var ratio: result<f64, error>`,
  `ratio /={zero: .error} step` assigns the checked quotient, a zero
  divisor's Err included. Its left operand is read as any operand is (the
  language's ERR-15): a place that holds an Err raises it there, and is
  not divided. For a place that holds the plain type, `.error` has no
  candidate (`operator.no_candidate`).

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

`seen += ["x", "y"]` there would put two entries, as `extend` does.
Deliberate source rejections: `count /= 2` for `var count: i64`
(`operator.no_candidate`); `fixed += 1` for `let fixed = 0`
(`binding.immutable`).
