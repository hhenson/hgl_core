# Documenting core

Status: agreed that every exported declaration of core is documented, to a
high standard, and that existing comments are brought to the language's
present form. DOC-1 to DOC-11 are that standard, proposed until reviewed.
No declaration of `hgl/` has a documentation comment yet.

Core's reference is generated from these comments. A reader meets a
declaration there first, and often there only: the comment has to be
enough to use it rightly without opening `spec/`.

The form is the language's
([source documentation](https://github.com/hhenson/hgl_spec/blob/main/spec/language/source-documentation.md)):
a `/** ... */` before the declaration, a summary, sections, and Markdown
text.

## What is documented

- **DOC-1.** Every exported operator, function, constant, struct and each
  of its fields, enum and each of its members, and native type has a
  comment. A candidate (`impl`) has one where something is particular to
  it, and says only that: the operator's comment is the contract of every
  candidate (SDOC-4).
- **DOC-2.** A comment is written from the declaration's page in `spec/`
  and says nothing the page does not. Where the page is silent or wrong,
  the page is corrected first.

## What a comment holds

- **DOC-3.** The summary is one sentence that says what the declaration
  gives, in the caller's terms. It does not restate the name. An operator
  that a symbol stands for shows the symbol.
- **DOC-4.** After the summary comes what a caller must know to use it
  rightly: when its result ticks and what it needs valid; what a null, a
  NaN, an empty collection or a zero does; what it leaves to another
  declaration. Short paragraphs, with no history and nothing of how it is
  built.
- **DOC-5.** `Type Args:` and `Args:` name every generic parameter,
  parameter and option, each with what it means; an option says what it
  is when none is given. `Returns:` says what the result is. `Raises:`
  names each code and when it is raised. A temporal declaration has
  `Ticks:` and `Validity:`.
- **DOC-6.** `Examples:` has at least one example that runs (SDOC-9): the
  common use first, then the case most likely to surprise. A temporal
  declaration is shown with an `eval` and its trace, and what is asked
  inside a node with a helper the example declares for itself. An
  example marked `no-run` is one that cannot run in a test, and the text
  beside it says why.
- **DOC-7.** A related declaration is named in backticks, which links it
  (SDOC-8): the inverse, the checked form, the one to use in its place.
- **DOC-8.** What is a formula is written as one, and a flow that words
  make hard is drawn (SDOC-7). Neither is decoration.
- **DOC-9.** A last section, `Rules:`, lists the rule IDs of core and of
  the language that the declaration rests on.
- **DOC-10.** Plain words, in the present tense, to the caller. A
  connection ticks. Nothing needs core's history to follow.

## Test cases

- **DOC-11.** A case's comment states the observable rule in its first
  sentence, as `tests/conformance/README.md` says, and ends with two
  sections, `Rules:` and `Status:`, in place of the two fields written
  `:Rules:` and `:Status:`.

## Worked examples

Each is a declaration of core as its comment would stand. A fence inside
a comment is of backticks or of tildes; these are of tildes because this
page's own fences are of backticks.

An operator:

```hgl
/**
Convert a value to another type: `convert<f64>(count)`.

Nothing changes a value's type without it, but an integer literal written
where an `f64` is expected. A conversion that cannot fail gives the new
value. One that can, a `str` that writes no number among them, gives a
`result`, or with `invalid: .fail` the value itself and an error that
escapes where it fails.

Type Args:
    T: The type to convert to. The call writes it.
    V: The type of the value given. It is inferred.
    O: The result's type: `T`, or `result<T, error>` where the conversion
        can fail.

Args:
    value: The value to convert.

Returns:
    The value as a `T`. The table of conversions in
    `spec/scalar-operators.md` says what each yields.

Raises:
    convert.invalid: Under `invalid: .fail`, a value that has no `T`.

Examples:
    ~~~hgl
    assert convert<f64>(3) == 3.0
    assert convert<str>(42) == "42"
    assert convert<bool>(0) == false
    ~~~

Rules:
    OPS-4, OPS-6, OPS-7, OPS-8
*/
export operator convert<T, V, O>(value: V) -> O
    requires O in {T, result<T, error>}
```

A constant:

```hgl
/**
The least by which one cycle's evaluation time follows another's: a microsecond.

`earliest_start` is this much after `never`, and `latest_end` this much
before `forever`.

Examples:
    ~~~hgl
    assert smallest_step == 1us
    assert earliest_start - never == smallest_step
    ~~~

Rules:
    CRUN-3
*/
export const smallest_step: duration = 1us
```

A function asked of an endpoint, whose example declares the node that
asks it:

```hgl
/**
Whether an endpoint holds a value.

An input is valid from its first tick and stays valid through the cycles
in which it does not tick, until it is invalidated. `valid` takes the
endpoint itself and reads nothing of it, so it is asked of an input that
has no value yet. `all_valid` asks it of a collection and each of its
children too, and `modified` asks whether the endpoint ticked in this
cycle.

Type Args:
    T: The type the endpoint carries.

Args:
    input: The endpoint asked about: an input, or a child of one.

Returns:
    True where the endpoint holds a value. For a window, true while it
    holds one. For a `ref` parameter, true once it has been given a
    designation, whatever its target holds.

Examples:
    `value` is silent, then 7, then silent, while `probe` ticks in each
    cycle.

    ~~~hgl
    fn held(value: i64, probe: bool) -> bool {
        when valid([]) { return valid(value) }
    }
    assert eval held(value: [_, 7, _], probe: [true, true, true]) == [false, true, true]
    ~~~

Rules:
    END-1, END-4, END-7, END-9, END-10
*/
export native const fn valid<T>(input: endpoint<T>) -> bool
```

A case:

```hgl
/**
A scalar becomes valid on its first tick and stays valid through silence.
A separate probe wakes the observer before and after that tick.

Rules:
    END-1, TIME-3

Status:
    agreed
*/
test silence_preserves_validity {
    assert eval watched_valid(
                watched: [_, 7, _, _],
                probe: [true, true, true, true]) == [false, true, true, true]
}
```
