# Core conformance tests

These HGL cases check that core's own code conforms to core's contract: its
operators, its portable bodies and its eval harness. They are test code only.
They form a module of their own, `core_conformance`, so that they use core as a
program does; nothing of them ships, and no program imports them. A struct or
a candidate declared here is a fixture of a test, never part of core: core's
source is `hgl/`.

| Part | What it checks |
| --- | --- |
| `scalar_operator_wrappers` | Operand order, numeric precision, tolerance options, special values and text repeated a count |
| `identity_arithmetic` | Addition and subtraction under the identity policy, ordinary lifting and generic candidates |
| `lists` | Append, extend, independence of an earlier list value, and an index outside a list |
| `join` | Separator configuration in the list and positional-pack forms |
| `mapping` | Fixed-size and keyed mapping, callback arguments and lifetimes, fixed-size reduction and growing ordered folds |
| `dense_traces` | Empty frames, leading and trailing silence, and recording through the last publication |
| `recorded_ticks` | Absolute recording times for published nullable deltas |
| `record_eval` | Empty recordings and retention of earlier recorded deltas |
| `conversions` | Each conversion: rounding, a number as text and text as a number, and what fails |
| `formatting` | Direct formatting and the formatting reference examples |
| `structural_conversion` | Collection text, component conversions and iteration order |
| `output` | Output to each process stream and publications from a writing handler |
| `diagnostics` | Value-code diagnostic bytes, explicit marks, wall time and handler publications |
| `tenors` | Tenor arithmetic, text, calendar extension and rolling rules |
| `zoned_datetimes` | A duration added to a zoned datetime, and equality by instant and zone or as UTC |

Run `hgl test hgl tests/core_conformance --target hgraph` from the core checkout
with that target installed. Shared runtime observations live in
[the conformance suite](../conformance/README.md). The output cases describe
the expected stream bytes for an external harness, as do the diagnostic
cases. A trace assertion in those cases observes the publications.
