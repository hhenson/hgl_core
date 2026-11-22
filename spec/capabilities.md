# Capabilities

Status: CAP-1 to CAP-8 are agreed.

## Scope and API

The [language temporal model](https://github.com/hhenson/hgl_spec/blob/main/spec/language/temporal-model.md)
says what a capability is: a node has one by `inject`, uses it in its hooks,
may give it to a parameter of its type, and does not retain it (TIME-8);
`run_config` is an application's and an integration test's, and no node's
(the language's RUN-6). This contract says what each one is: its type, its
read-only properties, and the operations on it.

| Capability | Type | Properties | Operations |
| --- | --- | --- | --- |
| `logger` | `Logger` | `level: LogLevel` | `log`, `log_debug`, `log_info`, `log_warning`, `log_error` |
| `clock` | `Clock` | `evaluation_time: datetime`, `now: datetime`, `cycle_time: duration`, `next_cycle_evaluation_time: datetime` | None |
| `scheduler` | `Scheduler` | `next_scheduled_time: datetime?`, `is_scheduled: bool`, `is_scheduled_now: bool` | `schedule`, `un_schedule`, `reset` |
| `alarm` | `Alarm` | None | `schedule` |
| `global_state` | `GlobalState` | None | `get`, `has`, `put` |
| `run_config` | `RunConfig` | `start: datetime?`, `end: datetime?`, `length: duration?` | None |

The types and the native operations are declared in
`hgl/native/capabilities.hgl`; `LogLevel`, the `log_*` helpers, and the
operators `schedule`, `get`, `has`, and `put` in `hgl/core/core.hgl`. Each
of the four is an operator: its name has a meaning beyond one capability,
and other candidates join it as they are declared. `get` already has the
dictionary's (the language's ERR-6).

## Agreed rules

- **CAP-1.** Each capability but `out` has a native type of core's module,
  named as the capability with a capital: `Logger`, `Clock`, `Scheduler`,
  `Alarm`, `GlobalState`, `RunConfig`. Core constructs none in HGL. A
  property is read as an ordinary value and is not assigned.
- **CAP-2.** `log(logger, level, message)` writes one record of `message` at
  `level`, a `LogLevel`: `debug`, `info`, `warning`, or `error`.
  `log_debug`, `log_info`, `log_warning`, and `log_error` each call `log` at
  their level. `logger.level` is the logger's current level.
- **CAP-3.** A clock has four properties: `evaluation_time`, `now`, and
  `next_cycle_evaluation_time`, each a `datetime`, and `cycle_time`, a
  `duration`. It has no operation.
- **CAP-4.** A scheduler has three properties: `next_scheduled_time`, a
  `datetime?`, and `is_scheduled` and `is_scheduled_now`, each a `bool`.
  `is_scheduled_now` is whether a wake asked of the scheduler is due in the
  current evaluation, whether or not an input has ticked in it, and it
  stays true for the rest of that evaluation after `un_schedule` or `reset`
  in it; the language's `scheduled()` is this property in a node that
  injects a scheduler (TIME-9). `schedule(scheduler, at)` asks
  for a wake: `at` is a `duration` or a `datetime`. It takes a `tag`, a
  `str?` that is null when omitted, and `on_wall_clock`, a `bool` that is
  false when omitted. `un_schedule(scheduler)` and
  `un_schedule(scheduler, tag: name)` cancel a pending wake, and
  `reset(scheduler)` every one. A wake asked for with a `duration` takes
  one more than zero, but in `start`, where `schedule(scheduler, 0s)` asks
  for a wake at the start, the node's first evaluation. A duration that is
  not admitted, a negative one in `start` included, fails the call (the
  language's TIME-8, which rejects one the compiler sees). One asked for at a
  `datetime` before the current evaluation time is dropped: there is no
  wake, and no failure. `on_wall_clock: true` times a wake on the wall
  clock, `clock.now`, in a run in real time. The time the request is
  measured from is then the later of the evaluation time and `clock.now`: a
  `duration` asks for a wake that long after it, and a `datetime` is a time
  on the wall clock. A wake of the option that is already due where it is
  asked for, at or before that time, is not dropped: it comes in the next
  cycle, at the later of that time and the evaluation time plus the
  engine's smallest step, and in `start` at that time. So a node that asks
  again and again for a wall-clock time just ahead keeps its wake where
  the clock passes the target first. In simulation the option changes
  nothing: the wake is timed on evaluation time, as one asked for without
  it, so that a graph runs the same in either. The failure's code is
  `schedule.non_positive_duration`; the language rejects a duration the
  compiler sees with `schedule.duration_bounds`. The
  [scheduling cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/scheduled_wakes.hgl)
  check the failure and the wall-clock option in simulation, and the
  [rejection cases](https://github.com/hhenson/hgl_core/blob/main/tests/conformance/rejections/scheduled_wakes.hgl)
  the rejection.
- **CAP-5.** An alarm is a cheaper scheduler: best effort, with nothing
  tracked, moved, or cancelled. `schedule(alarm, at)` asks for a wake, with
  `at` a `duration` or a `datetime`; asking again keeps the earliest. It has
  no property, no tag, no wall clock, and no cancellation. A node injects it
  in place of `scheduler`, with temporal inputs or none. It keeps no record
  of a wake: the language tells one from the node's inputs (TIME-9). What
  may be asked of it is what CAP-4 says may be asked of the scheduler,
  alike: a `duration` more than zero, or zero in `start`, which asks for a
  wake at the start, the node's first evaluation; a `datetime` before the
  current evaluation time is dropped.
- **CAP-6.** The global store is a run-wide store of ordinary values under
  string keys, each fixed when the graph is built (the language's FN-1).
  `get<V>(global_state, key)` returns `result<V, error>`. The call may specify
  V or infer it from its place. An absent key gives a failure. The place
  takes the result, raises the failure when it needs the value, or takes
  null in the nullable form (ERR-15). When V itself is nullable, a stored
  null is a success.
  `has(global_state, key)` is whether the key holds a value.
  `put(global_state, key, value)` puts one there. A key keeps the type of
  its first value: a `put` of another type is rejected where the two are in
  one program, with code `store.key_type` (the language's TIME-8), and fails
  otherwise. Reading or writing the store wakes no node and is no temporal
  input, so it orders no two nodes: a `put` is seen by what is evaluated
  after it, and the language's TIME-2 leaves the order of two nodes of one
  rank undefined. What one of them sees of the other's `put` in the same
  cycle is undefined. The key `"run.output"` is the run's (CRUN-8): a
  program that names it is rejected when its graph is built.
- **CAP-7.** `run_config` is the configuration of the run (CRUN-2), as it
  is supplied to the run (the language's RUN-6). It has three properties:
  `start` and `end`, each a `datetime?`, and `length`, a `duration?`. A
  field the configuration leaves out is null: `run_config.start` of a run
  given no start is null, and not the start the run resolves (CRUN-2). It
  has no operation. Who injects it, who supplies it, and where it is read
  are the language's RUN-6.

## What each member observes

- **CAP-8.** Each property and operation observes what this table gives.

| Member | Meaning |
| --- | --- |
| `clock.evaluation_time` | The logical time of the current cycle: the same for every node in it, and constant throughout it. In `start` it is the run's start time; `stop` retains the last cycle's |
| `clock.now` | The engine's estimate of wall-clock time: the computer's clock in UTC in real time, and evaluation time plus the cycle time in simulation. It may change between reads |
| `clock.cycle_time` | Real elapsed time since the current cycle began |
| `clock.next_cycle_evaluation_time` | The earliest time a following cycle could have: evaluation time plus the engine's smallest step. It asks for no cycle |
| `scheduler.is_scheduled` | True while a wake is pending |
| `scheduler.next_scheduled_time` | The time of the earliest pending wake; null with none |
| `schedule(scheduler, at)` | With a `duration`, a wake that long after the current evaluation time; with a `datetime`, a wake then |
| `un_schedule(scheduler)` | Cancels the earliest pending wake |
| `un_schedule(scheduler, tag: name)` | Cancels the wake asked for under `name` |
| `reset(scheduler)` | Cancels every pending wake |
| A wake asked for under a tag | Replaces the pending wake of that tag |
| `logger.level` | Records below it are dropped |

## Accepted illustration

```hgl
module examples.capability_types

const fn stamped(clock: Clock, value: f64) -> tuple<datetime, f64> =>
    (clock.evaluation_time, value)

fn stamp(value: f64) -> tuple<datetime, f64> {
    inject clock, logger
    when {
        log_info(logger, "stamped")
        return stamped(clock, value)
    }
}

fn quiet(value: f64) -> f64 {
    inject scheduler
    when {
        if scheduler.is_scheduled { un_schedule(scheduler) }
        return value
    }
}

fn counted(const every: duration = 1s) -> i64 {
    inject scheduler, global_state
    start {
        if !has(global_state, "count") { put(global_state, "count", 0) }
        schedule(scheduler, every, tag: "count")
    }
    when scheduled() {
        schedule(scheduler, every, tag: "count")
        put(global_state, "count", get<i64>(global_state, "count") + 1)
        return get<i64>(global_state, "count")
    }
}

fn woken(value: f64, const every: duration = 1s) -> f64 {
    inject alarm
    start { schedule(alarm, every) }
    when {
        schedule(alarm, every)
        return value
    }
}
```

`stamp` gives its clock to `stamped`, whose parameter is of the clock's type,
and logs through its logger. `quiet` reads a property of its scheduler and
cancels what is pending. `counted` wakes itself each second under a tag, and
counts in the global store under a key written out, which `start` gives a
value where it has none. `woken` is woken by its
input and by its alarm. Deliberate rejections: `clock.now = value`, a
property assigned; `Clock()`, a capability constructed;
`log(logger, "info", "text")`, a `str` where a `LogLevel` is taken;
`schedule(alarm, every, tag: "poll")`, a tag on an alarm; and
`put(global_state, 7, 1)`, a key that is no string. Conformance is planned.
