# Capabilities

Status: CAP-1 to CAP-6 are agreed. The meanings under "Proposed behaviour"
are proposed.

## Scope and API

The [language temporal model](https://github.com/hhenson/hgl_spec/blob/main/spec/language/temporal-model.md)
says what a capability is: a node has one by `inject`, uses it in its hooks,
may give it to a parameter of its type, and does not retain it (TIME-8). This
contract says what each one is: its type, its read-only properties, and the
operations on it.

| Capability | Type | Properties | Operations |
| --- | --- | --- | --- |
| `logger` | `Logger` | `level: LogLevel` | `log`, `log_debug`, `log_info`, `log_warning`, `log_error` |
| `clock` | `Clock` | `evaluation_time: datetime`, `now: datetime`, `cycle_time: duration`, `next_cycle_evaluation_time: datetime` | None |
| `scheduler` | `Scheduler` | `next_scheduled_time: datetime?`, `is_scheduled: bool` | `un_schedule`, `reset` |
| `alarm` | `Alarm` | None | None declared yet |
| `global_state` | `GlobalState` | None | None declared yet |

The types and the native operations are declared in
`hgl/native/capabilities.hgl`; `LogLevel` and the `log_*` helpers in
`hgl/core/core.hgl`. Asking for a wake, on a scheduler or an alarm, and the
global store's operations are not declared: their spellings are open.

## Agreed rules

- **CAP-1.** Each capability but `out` has a native type of core's module,
  named as the capability with a capital: `Logger`, `Clock`, `Scheduler`,
  `Alarm`, `GlobalState`. Core constructs none in HGL. A property is read as
  an ordinary value and is not assigned.
- **CAP-2.** `log(logger, level, message)` writes one record of `message` at
  `level`, a `LogLevel`: `debug`, `info`, `warning`, or `error`.
  `log_debug`, `log_info`, `log_warning`, and `log_error` each call `log` at
  their level. `logger.level` is the logger's current level.
- **CAP-3.** A clock has four properties: `evaluation_time`, `now`, and
  `next_cycle_evaluation_time`, each a `datetime`, and `cycle_time`, a
  `duration`. It has no operation.
- **CAP-4.** A scheduler has two properties: `next_scheduled_time`, a
  `datetime?`, and `is_scheduled`, a `bool`. `un_schedule(scheduler)` and
  `un_schedule(scheduler, tag: name)` cancel a pending wake, and
  `reset(scheduler)` every one. A wake may be asked for under a tag, a `str`.
- **CAP-5.** An alarm is a cheaper scheduler: best effort, with nothing
  tracked, moved, or cancelled. It has no property, no tag, and no
  cancellation. A node injects it in place of `scheduler`, with temporal
  inputs or none.
- **CAP-6.** The global store is a run-wide store of ordinary values. Reading
  or writing it wakes no node and is no temporal input.

## Proposed behaviour

What each property and operation observes, from the engine's contract. None is
a requirement until agreed.

| Member | Proposed meaning |
| --- | --- |
| `clock.evaluation_time` | The logical time of the current cycle: the same for every node in it, and constant throughout it. In `start` it is the run's start time; `stop` retains the last cycle's |
| `clock.now` | The engine's estimate of wall-clock time: the computer's clock in UTC in real time, and evaluation time plus the cycle time in simulation. It may change between reads |
| `clock.cycle_time` | Real elapsed time since the current cycle began |
| `clock.next_cycle_evaluation_time` | The earliest time a following cycle could have: evaluation time plus the engine's smallest step. It asks for no cycle |
| `scheduler.is_scheduled` | True while a wake is pending |
| `scheduler.next_scheduled_time` | The time of the earliest pending wake; null with none |
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
```

`stamp` gives its clock to `stamped`, whose parameter is of the clock's type,
and logs through its logger. `quiet` reads a property of its scheduler and
cancels what is pending. Deliberate rejections: `clock.now = value`, a
property assigned; `Clock()`, a capability constructed; and
`log(logger, "info", "text")`, a `str` where a `LogLevel` is taken.
Conformance is planned.
