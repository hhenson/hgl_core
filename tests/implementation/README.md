# Core implementation tests

These HGL cases check the behavior of core's operators, portable bodies and eval harness.

| Part | What it checks |
| --- | --- |
| `scalar_operator_wrappers` | Operand order, numeric precision, tolerance options and special values |
| `identity_arithmetic` | Addition and subtraction under the identity policy, ordinary lifting and generic candidates |
| `lists` | Append, extend and independence of an earlier list value |
| `dense_traces` | Empty frames, leading and trailing silence, and recording through the last publication |
| `recorded_ticks` | Absolute recording times for published nullable deltas |
| `record_eval` | Empty recordings and retention of earlier recorded deltas |
| `conversions` | Each conversion: rounding, a number as text and text as a number, and what fails |
| `tenors` | Tenor arithmetic, text, calendar extension and rolling rules |
| `zoned_datetimes` | A duration added to a zoned datetime, and equality by instant and zone or as UTC |

Run `hgl test hgl tests/implementation --target hgraph` from the core checkout
with that target installed. Shared runtime observations live in
[the conformance suite](../conformance/README.md).
