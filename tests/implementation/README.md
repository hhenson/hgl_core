# Core implementation tests

These HGL cases check the behavior of core's operators, portable bodies and eval harness.

| Part | What it checks |
| --- | --- |
| `scalar_operator_wrappers` | Operand order, numeric precision, tolerance options, special values and text repeated a count |
| `identity_arithmetic` | Addition and subtraction under the identity policy, ordinary lifting and generic candidates |
| `lists` | Append, extend, independence of an earlier list value, and an index outside a list |
| `dense_traces` | Empty frames, leading and trailing silence, and recording through the last publication |
| `recorded_ticks` | Absolute recording times for published nullable deltas |
| `record_eval` | Empty recordings and retention of earlier recorded deltas |
| `conversions` | Each conversion: rounding, a number as text and text as a number, and what fails |
| `formatting` | Direct formatting and the formatting reference examples |
| `structural_conversion` | Collection text, component conversions and iteration order |
| `output` | Output to each process stream and publications from a writing handler |
| `tenors` | Tenor arithmetic, text, calendar extension and rolling rules |
| `zoned_datetimes` | A duration added to a zoned datetime, and equality by instant and zone or as UTC |

Run `hgl test hgl tests/implementation --target hgraph` from the core checkout
with that target installed. Shared runtime observations live in
[the conformance suite](../conformance/README.md). The output cases describe
the expected stream bytes for an external harness. A trace assertion in
those cases observes the publications.
