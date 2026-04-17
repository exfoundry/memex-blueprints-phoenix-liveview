---
name: GenServer Testing
description: Patterns for testing GenServer processes — supervised startup, exit detection, and message synchronisation
type: reference
---

# GenServer Testing

No project-specific GenServer conventions yet — this blueprint captures general testing patterns to avoid common mistakes.

## Starting processes

Always use `start_supervised!/1` — it guarantees cleanup between tests regardless of test outcome:

```elixir
pid = start_supervised!({MyGenServer, arg: :value})
```

Never call `GenServer.start_link/3` directly in tests — the process outlives the test and corrupts state for subsequent tests.

## Waiting for a process to exit

Never use `Process.sleep/1`. Use `Process.monitor/1` and assert on the `:DOWN` message:

```elixir
ref = Process.monitor(pid)
assert_receive {:DOWN, ^ref, :process, ^pid, :normal}
```

## Synchronising before asserting side effects

Never sleep to let a GenServer process its messages. Use `:sys.get_state/1` — it blocks until the process has handled all prior messages:

```elixir
_ = :sys.get_state(pid)
# now safe to assert on side effects
```

## Do

- Use `start_supervised!/1` for all process startup in tests
- Use `Process.monitor/1` + `assert_receive {:DOWN, ...}` to wait for exit
- Use `:sys.get_state/1` to synchronise before asserting on side effects

## Don't

- Don't call `GenServer.start_link/3` directly in tests
- Don't use `Process.sleep/1` to synchronise
- Don't poll with `Process.alive?/1`
