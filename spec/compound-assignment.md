# Compound assignment

Status: ASG-1 through ASG-7 and the identities' names are agreed, but for
each sentence that begins "Proposed:". The declarations are reference
declarations. `hgl/` declares none of the identities yet, and declares
`append` and `extend` as native functions of a list.

## Scope and API

The [language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/expressions-and-bindings.md)
says that a compound assignment binds an identity whose first parameter is
a `var`, that on a connection in composition it is a rebinding, that a
place in `out` is given as its endpoint, and in what order it is evaluated
(EXPR-18). This contract has the identities and core's candidates.

| Syntax | Identity | Default |
| --- | --- | --- |
| `a += b` | `iadd_(a, b)` | `a = a + b` |
| `a -= b` | `isub_(a, b)` | `a = a - b` |
| `a *= b` | `imul_(a, b)` | `a = a * b` |
| `a /= b` | `idiv_(a, b)` | `a = a / b` |
| `a //= b` | `ifloordiv_(a, b)` | `a = a // b` |
| `a %= b` | `imod_(a, b)` | `a = a % b` |

Reference declarations, of `iadd_` and of one division:

```hgl
export operator iadd_<L, R>(var lhs: L, rhs: R)

# ASG-3: what the symbol gives, assigned.
impl const fn iadd_<T, R>(var lhs: T, rhs: R)
    requires add_(T, R) -> T {
    lhs = lhs + rhs
}

# ASG-5: a place in `out`, by the candidate its type has for a `var`.
native impl const fn iadd_<T, R>(var lhs: endpoint<T>, rhs: R)
    requires iadd_(var T, R)

# ASG-6: text grows in place.
native impl const fn iadd_(var lhs: str, rhs: str)

# ASG-4: an entry is appended, a collection of the place's type extends it.
impl const fn iadd_<C, T>(var lhs: C, rhs: T)
    requires C in {list<T>, set<T>} { append(lhs, rhs) }
impl const fn iadd_<C, T>(var lhs: C, rhs: C)
    requires C in {list<T>, set<T>} { extend(lhs, rhs) }

export operator append<C, T>(var values: C, item: T)
    requires C in {list<T>, set<T>}
export operator extend<C, T>(var values: C, items: C)
    requires C in {list<T>, set<T>}
# Proposed: `extend`, and `iadd_` given a collection, take T from the
# membership of C alone, which rests on a proposed clause of the
# language's GEN-3.

# ASG-7: one candidate for each result type its symbol has, with that
# symbol's policies and default.
export operator ifloordiv_<L, R>(var lhs: L, rhs: R)
impl const fn ifloordiv_<const zero: ZeroPolicy = .fail>(var lhs: i64, rhs: i64)
    requires zero == .fail { lhs = lhs //{zero: zero} rhs }
impl const fn ifloordiv_<const zero: ZeroPolicy = .ieee>(var lhs: f64, rhs: f64)
    requires zero in {.ieee, .nan, .fail} { lhs = lhs //{zero: zero} rhs }
impl const fn ifloordiv_<const zero: ZeroPolicy>(var lhs: result<i64, error>, rhs: i64)
    requires zero == .error { lhs = lhs //{zero: .error} rhs }
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
  (`operator.no_candidate`), `/` giving an `f64`. The default carries no
  option. Proposed: where core's candidates of a symbol declare options,
  core declares the compound candidates that carry them, as ASG-7 does
  for `zero`: `day += by` and `day -= by` of a `date` and a `tenor` take
  `calendar` and `rolling_rule` (TEN-5, TEN-7).
- **ASG-4.** `+=` on a list or a set has two candidates. Given an entry,
  it is `append`. Given a collection of the place's own type, it is
  `extend`. `append` and `extend` are operators, each with a candidate
  for a `list<T>` ([list operations](list-operations.md)) and one for a
  `set<T>`. `append` puts a member a set has not and leaves a set that
  holds it as it was; `extend` puts each member the set has not. A set
  is assigned as LIST-3 says of a list: the call assigns the place, and a
  value taken from it before is as it was. Proposed: a fixed-size list,
  `list<T, N>`, is no target of either. Where the right operand's type fits both candidates and nothing
  ranks one first, the empty literal given to a list of lists,
  `rows += []` for a `list<list<i64>>`, or to a set of sets, the
  statement is ambiguous (`operator.ambiguous`), and `append(rows, [])` or
  `extend(rows, [])` says which. Core's two candidates are declared so
  that neither is the more specific for such an operand. Where the two
  give an inline literal a list and a set reading, `s += []` for a
  `set<list<i64>>`, the list reading is taken and warned of, as for any
  call (`operator.sequence_reading`, the language's GEN-5).
- **ASG-5.** For a place in `out`, core's candidate takes a `var`
  `endpoint<T>`, an output, under the operator's name for the parameter.
  It reads what the place holds, applies the candidate that the place's
  type has for a `var`, with the options given, and publishes the result
  to the place as an assignment publishes it (the language's TIME-7): a
  collection as its change from what the place held. So what `+=` is of
  a `var` it is of an output: a list or a set grows, text is appended to,
  and a type that declares `iadd_` alone has it. The language says where
  such a statement stands and what it rejects (EXPR-18). Proposed: the
  candidate declares no option of its own. An option given is one the
  place's type declares for its `var` candidate, and is given on to it:
  the statement is admitted where that configured call of the `var`
  candidate is.
- **ASG-6.** `text += piece` appends in place, at the cost of the piece
  (the language's TYPE-1).
- **ASG-7.** `/=`, `//=` and `%=` take `zero` as their symbol does
  (NUM-12). Core declares a candidate of each for each result type its
  symbol has, with the policies that give that type and the symbol's
  default for it: `.ieee` for an `f64`, `.fail` for an `i64` and a
  `duration`. `.error` gives a `result`, so its candidates take a place
  that holds one: for `var ratio: result<f64, error>`,
  `ratio /={zero: .error} step` assigns the checked quotient, a zero
  divisor's Err included. The right operand is evaluated first (the
  language's EXPR-18). The candidate then reads the place as an operand
  (the language's ERR-15): one that holds an Err raises it, and is not
  divided. For a place that holds the plain type, and for one that holds
  a nullable, `.error` has no candidate (`operator.no_candidate`).

## Examples

Accepted illustration, a fragment; conformance planned:

```hgl
struct Money { pence: i64 }
impl const fn add_(lhs: Money, rhs: Money) -> Money => Money(pence: lhs.pence + rhs.pence)

test compound_forms {
    var count = 7
    count += 1
    count //={zero: .fail} 3
    assert count == 2

    var seen: list<str> = ["a"]
    seen += "b"                     # append
    seen += ["c", "d"]              # extend
    assert seen == ["a", "b", "c", "d"]

    var marks: set<i64> = {1, 2}
    marks += 2                      # held already
    marks += {2, 3}
    assert marks == {1, 2, 3}

    var total = Money(pence: 5)
    total += Money(pence: 7)        # the default, from `add_`
    assert total == Money(pence: 12)

    var ratio: result<f64, error> = ok(8.0)
    ratio /={zero: .error} 2.0
    assert ratio == ok(4.0)
    ratio /={zero: .error} 0.0
    assert is_err(ratio)
}

fn running(price: f64) -> f64 {
    inject out
    when {
        if valid(out) { out += price }
        else { out = price }
    }
}

test output_accumulates {
    assert eval running(price: [1.0, _, 2.5]) == [1.0, _, 3.5]
}
```

Deliberate source rejections, each where it is written: `count /= 2` for
`var count: i64` (`operator.no_candidate`); `rows += []` for
`var rows: list<list<i64>>` (`operator.ambiguous`);
`price /={zero: .error} 2.0` for `var price: f64`
(`operator.no_candidate`); `fixed += 1` for `let fixed = 0`
(`binding.immutable`); `iadd_([1], 2)`, whose first argument is no
place (`argument.place`).
