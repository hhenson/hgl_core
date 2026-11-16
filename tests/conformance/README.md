# Runtime conformance cases

Status: fixtures cover agreed and proposed behavior. The eval contract and
its runner remain pending.

Each file is a part of the test-only module `conformance` (CONF-3). Fixtures
are private functions that publish or wire their own output.

| Part | Test | Rules observed |
| --- | --- | --- |
| `eval_recording` | `recording_silent_ticks` | CEVAL-7/8/10 (agreed), TST-6 |
| `eval_recording` | `recording_empty_handoff` | CEVAL-4/9/10 (agreed), TST-6 |
| `eval_recording` | `recording_preserves_null_publications` | CEVAL-7, END-1/3 (agreed), TST-6 |
| `eval_recording` | `recording_store_is_private` | CEVAL-6/9 (agreed) |
| `eval_recording` | `recording_result_outlives_run` | CEVAL-4/5/10 (agreed), TST-6 |
| `identity_arithmetic` | `identity_arithmetic_publications` | OPS-9, END-1, END-5 |
| `identity_arithmetic` | `identity_arithmetic_invalidates_and_recovers` | OPS-9, END-4 |
| `identity_arithmetic` | `repeated_invalidation_is_silent` | OPS-9, END-4 |
| `identity_arithmetic` | `unconfigured_arithmetic_uses_lifting` | OPS-9, FN-6, FN-11 |
| `identity_arithmetic` | `identity_arithmetic_is_generic` | OPS-9, GEN-7, GEN-16 |
| `scalar_eval` | `equal_publications_remain_ticks` | CEVAL-3, END-2 (agreed), TST-6 |
| `scalar_eval` | `replay_preserves_equal_ticks` | CEVAL-3 (agreed), TST-6 |
| `scalar_eval` | `all_silent_input_is_empty` | CEVAL-2 (agreed), TST-6 |
| `scalar_eval` | `empty_input_remains_empty` | CEVAL-4 (agreed), TST-6, TST-8 |
| `scalar_eval` | `leading_silence_locates_a_tick` | CEVAL-2 (agreed), TST-6 |
| `scalar_eval` | `zero_false_and_empty_text_are_present` | CEVAL-3, END-1, TST-9 |
| `scalar_eval` | `dense_result_ends_at_last_publication` | END-3, TIME-3, TST-6 |
| `scalar_eval` | `runs_are_independent` | CEVAL-5 (agreed), TIME-5, TST-6, TST-8 |
| `mixed_eval` | `mixed_replay` | CEVAL-2 (agreed), TST-6, TST-9. Set delta profile is proposed. |
| `record_eval` | `record_retains_each_delta` | CEVAL-4 (agreed), TST-6. Set delta profile is proposed. |
| `record_eval` | `record_can_be_empty` | CEVAL-4 (agreed), TST-6. Set delta profile is proposed. |
| `failure_eval` | `eval_preserves_failure` | CEVAL-6 (agreed), ERR-5, NUM-9, TST-10 |
| `scalar_endpoints` | `silence_preserves_validity` | END-1 (agreed), TIME-1, TIME-3 |
| `scalar_endpoints` | `equal_publication_is_modified_and_idle_is_not` | END-2 (agreed), TIME-3 |
| `scalar_endpoints` | `retained_value_is_read_without_a_new_tick` | END-1, FN-7, TIME-3 |
| `scalar_endpoints` | `publish_invalidate_recover` | END-4 (agreed), TIME-3, TST-6 |
| `scalar_endpoints` | `publication_establishes_validity` | END-1 (agreed), TIME-3, TST-6 |
| `scalar_endpoints` | `equal_publications_are_modified` | END-2 (agreed), TIME-3, TST-6 |
| `set_endpoints` | `set_writes_coalesce` | END-5 (agreed), TIME-2, TST-6. Set delta syntax and semantics are proposed. |

CEVAL, CONF, END, NUM, and OPS rules are in core's `spec/`. ERR, FN, GEN, TIME, and TST
rules are in the [language specification](https://github.com/hhenson/hgl_spec).
