# Temporal addition and subtraction

Status: agreed same-type identity handling (OPS-9). The HGL bodies are generic;
native operations supply the ordinary value arithmetic. Runtime execution
still needs a target adapter and eval runner.

The `invalid` option is a const generic. `invalid: .identity` admits one valid
operand. Omitting it keeps ordinary operator selection and lifting. These
temporal candidates require the explicit option and `add_(T, T) -> T`, or
`sub_(T, T) -> T` and `neg_(T) -> T`. They invent no identity value or conversion.

Their guarded `when` clauses are in source order. The first satisfied condition
runs its body; `return` exits the node's current evaluation. The final
`valid([])` clause is reached only when neither operand is valid (TIME-3, FN-7).

## Wired example

`produce` publishes each non-null command, including zero. A null command
invalidates its output. `L` and `R` are the two producer outputs wired into
each arithmetic node. `V` observes the sum's validity, including invalidation.

```hgl
module examples.identity_arithmetic

fn produce(command: i64?) -> i64 {
    inject out
    when valid(command) {
        if command != null { return command }
        invalidate(out)
    }
}

fn sum(left: i64?, right: i64?) -> i64 =>
    produce(left) +{invalid: .identity} produce(right)

fn difference(left: i64?, right: i64?) -> i64 =>
    produce(left) -{invalid: .identity} produce(right)

fn watch(value: i64) -> bool {
    when valid([]) { return valid(value) }
}

fn sum_validity(left: i64?, right: i64?) -> bool => watch(sum(left, right))

test identity_arithmetic {
    assert eval(sum,
        left:  [7, _, null, _, 9, _, _, 0, _],
        right: [_, 2, _, null, _, _, 3, _, null]) ==
        [7, 9, 2, _, 9, _, 12, 3, 0]
    assert eval(difference,
        left:  [7, _, null, _, 9, _, _, 0, _],
        right: [_, 2, _, null, _, _, 3, _, null]) ==
        [7, 5, -2, _, 9, _, 6, -3, 0]
    assert eval(sum_validity,
        left:  [7, _, null, _, 9, _, _, 0, _],
        right: [_, 2, _, null, _, _, 3, _, null]) ==
        [true, true, true, false, true, _, true, true, true]
}
```

`t1` is the runner's origin; successive rows are one minimum engine step apart.
Both producers finish their writes before the arithmetic node runs. `_` is no
publication; invalidation is a state change, observed by `V` at `t4`.

| Tick | Left command | Right command | `L` retained | `R` retained | Sum tick | Difference tick | `V` tick |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `t1` | `7` | `_` | `7` | Invalid | `7` | `7` | `true` |
| `t2` | `_` | `2` | `7` | `2` | `9` | `5` | `true` |
| `t3` | `null` | `_` | Invalid | `2` | `2` | `-2` | `true` |
| `t4` | `_` | `null` | Invalid | Invalid | Invalidate | Invalidate | `false` |
| `t5` | `9` | `_` | `9` | Invalid | `9` | `9` | `true` |
| `t6` | `_` | `_` | `9` | Invalid | `_` | `_` | `_` |
| `t7` | `_` | `3` | `9` | `3` | `12` | `6` | `true` |
| `t8` | `0` | `_` | `0` | `3` | `3` | `-3` | `true` |
| `t9` | `_` | `null` | `0` | Invalid | `0` | `0` | `true` |
