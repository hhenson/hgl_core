# Compound assignment

Status: ASG-1 through ASG-4, ASG-6 and the identities' names are agreed.
ASG-5, ASG-7 and the declarations below are proposed. `hgl/` declares
none of them yet.

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

# ASG-4: one more entry, and every entry of another collection.
native impl const fn iadd_<C, T>(var lhs: C, rhs: T)
    requires C in {list<T>, set<T>}
native impl const fn iadd_<C, T>(var lhs: C, rhs: C)
    requires C in {list<T>, set<T>}

# ASG-4: the two named, each the `+=` it names.
export operator append<C, T>(var collection: C, item: T)
    requires C in {list<T>, set<T>}
export operator extend<C, T>(var collection: C, items: C)
    requires C in {list<T>, set<T>}
impl const fn append<C, T>(var collection: C, item: T)
    requires C in {list<T>, set<T>} { collection += item }
impl const fn extend<C, T>(var collection: C, items: C)
    requires C in {list<T>, set<T>} { collection += items }
```

## Rules

- **ASG-1.** Each compound assignment binds its own identity, and every
  value candidate of one takes its first parameter as a `var` (the
  language's FN-15): the call assigns the place, and gives no result.
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
  it puts that entry: after the last of a list, or into a set, which is
  left as it was where it holds the member. Given a collection of the
  place's own type, it puts each of its entries, a list's in order.
  `append` and `extend` are operators that name the two and add no
  behavior: each is the `+=` it names, and
  [list operations](list-operations.md) gives what a list holds
  afterwards. Proposed: where the right operand's type fits both, the
  empty literal given to a collection of collections, `rows += []` for a
  `list<list<i64>>`, the statement is ambiguous (`operator.ambiguous`),
  and `append(rows, [])` or `extend(rows, [])` says which.
- **ASG-5.** Proposed: for a place in `out`, core's candidate takes
  `endpoint<T>` and publishes what the default would assign, as an
  assignment to that place does (the language's TIME-7).
- **ASG-6.** `text += piece` appends in place, at the cost of the piece
  (the language's TYPE-1).
- **ASG-7.** Proposed: `/=`, `//=` and `%=` take `zero` as their symbol
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
