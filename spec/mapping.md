# Mapping and reduction

Status: MAP-1 through MAP-10, RED-1 through RED-7 and SET-1 through SET-2
are agreed. `hgl/` declares `map` and `reduce` and defines their fixed-size
candidates. The other candidates declare native implementation obligations.
The properties of `add_` and `mul_` are declared under RED-7.

## Scope and API

`map` applies a function to each key of dictionaries, or to each position
of lists, as its own child graph. `reduce` combines a collection's members
into one value. The
[language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/functions-and-phases.md)
says how a parameter takes a function (FN-17), how a pack is declared for
that function's call (FN-8), how a declaration takes configuration in
braces (FN-18), and how a pack is read, made by a loop and given to a call
(FN-19). A loop makes no child for a member that comes and goes
while the program runs (CTRL-7): these operators do.

| Core API | Contract |
| --- | --- |
| `map{key, join, keys}(f, args...)` | One child of `f` for each key, or each position |
| `reduce{associative, commutative}(f, items, zero)` | The members of `items` combined by `f` |
| `keys(dictionary)` | The set of a dictionary's keys |
| `union(a, b)`, `intersection(a, b)` | The members either set has, and those both have |

```hgl
export enum Join { inner, outer, left }

# The operator: a function, then that function's own arguments.
export operator map<R, C, ...Ps, ...As, const key: str = "key">{
    const join: Join = .inner,
}(f: fn(...{Ps}) -> R, args: ...{As for f without key}) -> C

# Keyed: at least one argument is a dictionary of its parameter's type,
# and every argument is that or its parameter's own type.
native impl fn map<K, R, ...Ps, ...As, const key: str = "key">{
    const join: Join = .inner,
    keys: set<K>? = null,
}(f: fn(...{Ps}) -> R, args: ...{As for f without key}) -> dict<K, R>
    requires any arg in arg_keys(As) { As[arg] == dict<K, Ps[arg]> }
          && each arg in arg_keys(As) { As[arg] in {Ps[arg], dict<K, Ps[arg]>} }

# By position: the same of a list.
native impl fn map<R, ...Ps, ...As, const key: str = "key">{
    const join: Join = .inner,
}(f: fn(...{Ps}) -> R, args: ...{As for f without key}) -> list<R>
    requires any arg in arg_keys(As) { As[arg] == list<Ps[arg]> }
          && each arg in arg_keys(As) { As[arg] in {Ps[arg], list<Ps[arg]>} }

# By position, over lists of one fixed size: written in HGL. One call of
# `f` for each position, each given a pack made from `f`'s own parameters.
impl fn map<R, N, ...Ps, ...As, const key: str = "key">{
    const join: Join = .inner,
}(f: fn(...{Ps}) -> R, args: ...{As for f without key}) -> list<R, N>
    requires any arg in arg_keys(As) { As[arg] == list<Ps[arg], N> }
          && each arg in arg_keys(As) { As[arg] in {Ps[arg], list<Ps[arg], N>} }
{
    return [for i in 0..<N:
        f(...{for arg in arg_keys(Ps):
            if arg == key { i }
            else if arg in arg_keys(As) { at_position<Ps[arg]>(args[arg], i) }})]
}

# One argument at one position: its entry where it is split, itself where
# it is whole. The candidates are selected by type. Not exported.
operator at_position<P, A>(arg: A, const at: i64) -> P
impl fn at_position<P, N>(arg: list<P, N>, const at: i64) -> P => arg[at]
impl fn at_position<P>(arg: P, const at: i64) -> P => arg

# The operator: a function of what is accumulated and a member.
export operator reduce<L, R, C,
    const associative: bool = associative(f),
    const commutative: bool = commutative(f),
>(f: fn(L, R) -> L, items: C, zero: L? = null) -> L

# Tree, over a list: the members in order, in any grouping.
native impl fn reduce<V, const associative: bool = associative(f)>(
    f: fn(V, V) -> V, items: list<V>, zero: V? = null) -> V
    requires associative

# Tree, over a set or a dictionary's values: in any order and grouping.
native impl fn reduce<V,
    const associative: bool = associative(f),
    const commutative: bool = commutative(f),
>(f: fn(V, V) -> V, items: set<V>, zero: V? = null) -> V
    requires associative && commutative

native impl fn reduce<K, V,
    const associative: bool = associative(f),
    const commutative: bool = commutative(f),
>(f: fn(V, V) -> V, items: dict<K, V>, zero: V? = null) -> V
    requires associative && commutative

# Ordered, over a list: a fold from the zero, which it requires.
native impl fn reduce<L, R, const associative: bool = associative(f)>(
    f: fn(L, R) -> L, items: list<R>, zero: L) -> L
    requires !associative

# Tree, over a list of a fixed size: written in HGL, from the first member.
impl fn reduce<V, N, const associative: bool = associative(f)>(
    f: fn(V, V) -> V, items: list<V, N>, zero: V? = null) -> V
    requires associative && N != 0
{
    var acc = items[0]
    for i in 1..<N { acc = f(acc, items[i]) }
    return acc
}

# Ordered, over a list of a fixed size: written in HGL, from the zero.
impl fn reduce<L, R, N, const associative: bool = associative(f)>(
    f: fn(L, R) -> L, items: list<R, N>, zero: L) -> L
    requires !associative
{
    var acc = zero
    for item in items { acc = f(acc, item) }
    return acc
}

# The laws `reduce` reads (the language's GEN-9), added to the operators
# core has.
export operator add_<L, R, O>(lhs: L, rhs: R) -> O
    properties<i64> { associative, commutative, identity = 0 }
    properties<f64> { associative, commutative, identity = 0.0 }
export operator mul_<L, R, O>(lhs: L, rhs: R) -> O
    properties<i64> { associative, commutative, identity = 1 }
    properties<f64> { associative, commutative, identity = 1.0 }

export operator keys<K, V>(dictionary: dict<K, V>) -> set<K>
export operator union<T>(a: set<T>, b: set<T>) -> set<T>
export operator intersection<T>(a: set<T>, b: set<T>) -> set<T>
```

The candidates over lists of a fixed size are written in HGL, and the
rest are native. `at_position` is core's own: an operator is private unless
exported (the language's UNIT-3).

## Rules

### Mapping

- **MAP-1.** `map` has three candidates. In the keyed form every argument
  that is split is a dictionary, and the result is a `dict<K, R>`. In the
  form by position every argument that is split is a list, and the result
  is a `list<R>`; where the lists are of one fixed size, `list<P, N>`, the
  result is a `list<R, N>`. A call that splits two of those kinds has no
  candidate. `keys` is the keyed form's alone. The candidate over lists
  of a fixed size is written in HGL: it calls `f` once for each position,
  where the graph is built.
- **MAP-2.** The arguments after `f` are those of `f`'s own call,
  positional or named (the language's FN-8). One whose type is its
  parameter's type is given whole to every child. One that is a
  `dict<K, P>`, a `list<P>`, or a `list<P, N>`, for a parameter of type `P` is split: each
  child reads the entry of its own key, or position. A generic `f` is
  given with its generic arguments: the function type takes any signature
  and decides none of them (FN-17).
- **MAP-3.** At least one argument is split, which each candidate's `any`
  requires. A call in which every argument is whole has no candidate,
  with `keys` given or not.
- **MAP-4.** The key type K is the member type of `keys`, where `keys` is
  given. Otherwise it is the key type of the first split dictionary, in the
  order of `f`'s parameters: the first argument the candidate's `any`
  holds for. Every split dictionary has that key type, and one that has
  another has no candidate.
- **MAP-5.** In the keyed form the key set is `keys`, where given.
  Otherwise it is the join of the split dictionaries' key sets: `.inner`,
  the keys every one has; `.outer`, the keys any has; `.left`, the keys of
  the first, in the order of `f`'s parameters. The default is `.inner`.
- **MAP-6.** In the form by position the positions are `0` to `n - 1`,
  where `n` is the join of the split lists' lengths: `.inner`, the
  shortest; `.outer`, the longest; `.left`, the first's. Lists of a fixed
  size share one `N`, which is `n`: the join has nothing to choose.
- **MAP-7.** There is one child of `f` for each key in the key set, or each
  position. A child starts in the cycle its key enters and stops in the
  cycle it leaves. A key that leaves and enters again in a later cycle has
  a new child, with fresh state. A key that leaves and enters again within
  one cycle keeps its child and its state. An update of an entry keeps the
  child. Over lists of a fixed size the `N` children are made where the
  graph is built and live as long as it does.
- **MAP-8.** A child whose key, or position, a split argument has no entry
  for sees that input as not valid, until the entry arrives. This arises
  under `.outer`, `.left` and `keys`, and never under `.inner`.
- **MAP-9.** Where `f` has a parameter of the name `key` gives, `"key"`
  unless the call says otherwise, that parameter is the child's key: a
  connection of K, or of `i64` for a position, which publishes once, in
  the cycle the child starts. No argument is given for it.
- **MAP-10.** The result has one entry for each child, which holds what the
  child publishes. An entry is valid once its child has published, and is
  removed in the cycle its child stops. Where `f` gives no result, `map`
  gives none. A failure that leaves a child is a failure of the `map`, as
  one that leaves any node of its graph is.

### Reduction

- **RED-1.** `items` is a `list<R>`, a `list<R, N>`, a `set<R>`, or a
  `dict<K, R>`, whose values are what is reduced. `f` combines what has
  been accumulated, an `L`, with a member, an `R`, and gives an `L`.
- **RED-2.** Only members that are valid are reduced. With none, the result
  is `zero`, where given, and is otherwise not valid. Over a list of a
  fixed size every position is reduced, and the result is valid once
  every position is: a function that takes and gives nullable values
  reduces one whose positions come and go.
- **RED-3.** The tree form combines the members in whatever grouping the
  implementation chooses. Its candidates take a function of one type,
  `fn(V, V) -> V`, and require `associative`. That option is
  `associative(f)` unless the call gives it: true where `f` is an operator
  that declares `associative` for V, and false for any other function. A
  call says `{associative: true}` of a function that declares nothing, and
  `{associative: false}` to take the ordered form whatever `f` declares.
  Over a list of a fixed size the tree form folds in position order from
  the first member, and does not combine a `zero` (RED-6); a list of no
  positions has no first member, and its call rejects.
- **RED-4.** Over a `set` or a `dict`, which have no order, the
  candidates require `commutative` too, which is `commutative(f)` unless
  the call gives it. A call that meets neither has no candidate, and
  neither has one whose `L` and `R` differ: only the tree form reduces a
  `set` or a `dict`.
- **RED-5.** The ordered form is the candidate that requires
  `!associative`, over a list. It folds the valid positions in order, from
  `zero`: `f(f(f(zero, x0), x1), x2)`. Its `zero` has no default: a call
  that leaves it out rejects.
- **RED-6.** A `zero` given to the tree form is an identity of `f`:
  `f(zero, x)` and `f(x, zero)` are `x`. Whether the implementation
  combines it with the members is therefore not observable.
- **RED-7.** Core declares `associative`, `commutative` and an identity for
  `add_` and for `mul_`, over `i64` and over `f64`. Over `f64` the grouping
  the tree form chooses can change the last digits of a result: the
  declaration is a contract (GEN-9), and a caller that needs one grouping
  asks for the ordered form.

### Key sets

- **SET-1.** `keys(dictionary)` is the set of the keys the dictionary has.
  As a connection it changes as keys are added and removed, and not when
  an entry's value changes.
- **SET-2.** `union(a, b)` has the members either set has, and
  `intersection(a, b)` those both have.

## Examples

Accepted illustration, complete module:

```hgl
module examples.mapping

fn scaled(price: f64, factor: f64) -> f64 => price * factor

fn labelled(key: str, price: f64) -> str => key

fn spread(bid: f64, ask: f64) -> f64 => ask - bid

fn all_scaled(prices: dict<str, f64>, factor: f64) -> dict<str, f64> =>
    map(scaled, prices, factor)

fn all_labelled(prices: dict<str, f64>) -> dict<str, str> =>
    map(labelled, prices)

fn spreads(bids: dict<str, f64>, asks: dict<str, f64>) -> dict<str, f64> =>
    map(spread, bids, asks)

fn wanted_spreads(bids: dict<str, f64>, asks: dict<str, f64>, wanted: set<str>) -> dict<str, f64> =>
    map{keys: wanted}(spread, bids, asks)

fn by_position(prices: list<f64>, factor: f64) -> list<f64> =>
    map(scaled, prices, factor)

fn three_scaled(prices: list<f64, 3>, factor: f64) -> list<f64, 3> =>
    map(scaled, prices, factor)

fn total(prices: dict<str, f64>) -> f64 => reduce(add_, prices, 0.0)

struct Event { size: i64 }

const fn applied(position: i64, event: Event) -> i64 => position + event.size

fn position_after(events: list<Event>) -> i64 => reduce(applied, events, 0)
```

In `all_scaled`, `prices` is a `dict<str, f64>` for the parameter `price`,
an `f64`, so it is split, and `factor` is given whole: there is a child of
`scaled` for each key of `prices` (MAP-2, MAP-7). `all_labelled` gives
`labelled` nothing for `key`, which is each child's key (MAP-9). `spreads`
has a child for each key both dictionaries have (MAP-5), and
`wanted_spreads` one for each member of `wanted`, where a child whose key
`asks` has not sees `ask` as not valid (MAP-8). `by_position` is the form
by position, and gives a `list<f64>` (MAP-1); `three_scaled` has its
three children from the start, and gives a `list<f64, 3>` (MAP-7). `total` is the tree form:
`add_` declares `associative` and `commutative` for `f64`, and `0.0` is its
identity (RED-3, RED-4, RED-6, RED-7). `position_after` is the ordered form:
`applied` takes an `i64` and an `Event`, so it folds the events in order
from 0 (RED-5).

Deliberate rejections: `map(scaled, factor: two, price: one)` with both
arguments an `f64`, which splits nothing (MAP-3); `map(spread, bids,
positions)` with `positions` a `list<f64>`, a dictionary and a list
(MAP-1); `map{keys: wanted}(scaled, items, factor)` with `items` a
`list<f64>`, since the form by position has no `keys`;
`reduce(applied, events)`, the ordered form with no `zero` (RED-5); and
`reduce(spread, prices, 0.0)`, where `spread` is not known to be
associative and `prices` is a dictionary (RED-4). No execution is claimed.
