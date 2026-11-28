# List operations

Status: LIST-1 through LIST-3 are agreed. Each operation is to become the
`+=` it names ([compound assignment](compound-assignment.md), ASG-4); `hgl/`
still declares the two as native functions.

## Scope and API

The [language specification](https://github.com/hhenson/hgl_spec/blob/main/spec/language/types-and-values.md)
says what a list is, how one is written out, and how an element of one is
read and assigned. This contract has the operations that give a list more
entries.

| Core API | Contract |
| --- | --- |
| `append<V>(var values: list<V>, item: V)` | `item` after the last entry of `values` |
| `extend<V>(var values: list<V>, items: list<V>)` | Each entry of `items`, in order, after the last entry of `values` |

Both are declared in `hgl/native/lists.hgl`.

## Rules

- **LIST-1.** `append(values, item)` gives the place `values` the list it
  held with `item` after its last entry. It returns nothing.
- **LIST-2.** `extend(values, items)` gives the place `values` the list it
  held with each entry of `items` after its last entry, in the order
  `items` has them. An empty `items` leaves it as it was. It returns
  nothing.
- **LIST-3.** Each takes its list as a `var` parameter (the language's
  FN-15). The call assigns the place: a value taken from the place before
  the call is as it was. A provider need not copy the list where nothing
  else holds it.

## Examples

Accepted illustration, complete module:

```hgl
module examples.lists

test a_list_grows {
    var entries: list<i64> = [1, 2]
    let before = entries
    append(entries, 3)
    extend(entries, [4, 5])
    assert entries == [1, 2, 3, 4, 5]
    assert before == [1, 2]
}

fn gathered(value: i64) -> list<i64> {
    cache seen: list<i64> = []
    when {
        append(seen, value)
        return seen
    }
}
```

`before` took the list when it had two entries, and has two still (LIST-3).
`gathered` keeps what it has seen in a `cache`, a place a `var` parameter
takes. Deliberate source rejections, each the language's: `append([1], 2)`,
whose list is no place (`argument.place`); and `append(fixed, 2)` for
`let fixed = [1]` (`binding.immutable`).
