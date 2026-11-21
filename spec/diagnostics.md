# Diagnostics

Status: DBG-1 is agreed. DBG-2 through DBG-4 are proposed. Nothing here is
declared in `hgl/` yet.

## Scope and API

`debug_print` is a tool for the person writing a graph: it shows what a
connection publishes while the program runs. It is meant to be put in, looked
at, and taken out again. A project may well refuse it in code it keeps.
Logging that stays is the logger's
([capabilities](capabilities.md)).

| Core API | Contract |
| --- | --- |
| `debug_print<T>(const label: str, value: T)` | One line on standard output for each publication of `value` |

```hgl
export native fn debug_print<T>(const label: str, value: T)
```

## Rules

- **DBG-1.** `debug_print(label, value)` is called where a graph is built.
  In each cycle in which `value` publishes it writes one line to the
  process's standard output, and it gives no result.
- **DBG-2.** The line has the label and then the value as text. Its exact
  form is the implementation's: it is for a person to read, and no program
  is to depend on it.
- **DBG-3.** `label` is fixed where the graph is built. `value` is of any
  type, and its publication is all that wakes the call: nothing is written
  before `value` first publishes, nor in a cycle in which it does not.
- **DBG-4.** It changes nothing else a program can observe. Where several
  calls write in one cycle, their lines stand in the order their nodes are
  evaluated, which the language orders by what depends on what and not
  otherwise.

## Examples

Accepted illustration, complete module:

```hgl
module examples.diagnostics

fn doubled(price: f64) -> f64 {
    debug_print("price", price)
    let twice = price * 2.0
    debug_print("twice", twice)
    return twice
}
```

Run with `price` publishing `1.5` and then `4.0`, `doubled` writes four
lines, `price` and then `twice` in each of the two cycles, since `twice`
depends on `price` (DBG-4), and gives what it gave without them. No
conformance case observes the lines: a trace does not see standard output.
