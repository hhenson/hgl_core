# Runtime conformance cases

Status: proposed. These cases observe proposed eval and endpoint rules. They
become conformance when those rules are agreed. No runner or adapter is
supplied and no conformance claim is made.

Each file is a part of the test-only module `conformance` (CONF-3). Fixtures
are private functions that publish their own output.

| Part | Test | Rules observed |
| --- | --- | --- |
| `scalar_eval` | `equal_publications_remain_ticks` | CEVAL-3, END-2, TST-6 |
| `scalar_eval` | `silent_horizon_survives` | CEVAL-5, TST-6 |
| `scalar_eval` | `empty_input_remains_empty` | CEVAL-4, TST-6, TST-8 |
| `scalar_eval` | `leading_and_trailing_silence_survive` | CEVAL-5, TST-6 |
| `scalar_eval` | `zero_false_and_empty_text_are_present` | CEVAL-3, END-1, TST-9 |
| `scalar_eval` | `horizon_is_the_longest_input` | END-3, TIME-3, TST-6 |
| `scalar_eval` | `runs_are_independent` | CEVAL-5, TIME-5, TST-8 |
| `scalar_endpoints` | `silence_preserves_validity` | END-1, TIME-1, TIME-3 |
| `scalar_endpoints` | `equal_publication_is_modified_and_idle_is_not` | END-2, TIME-3 |
| `scalar_endpoints` | `retained_value_is_read_without_a_new_tick` | END-1, FN-7, TIME-3 |

CEVAL, CONF, and END rules are in this repository's `spec/`. FN, TIME, and TST
rules are in the [language specification](https://github.com/hhenson/hgl_spec).
