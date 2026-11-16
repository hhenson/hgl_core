# Core implementation tests

`scalar_operator_wrappers.hgl` checks both operand orders, exact mixed numeric
comparison, tolerance forwarding, and NaN/signed-zero behavior (NUM-1, NUM-2,
NUM-4, NUM-5, OPS-1). These are value tests; they need no temporal eval.

Source checking does not execute them. A native provider and test execution
target are still required to run the assertions.
