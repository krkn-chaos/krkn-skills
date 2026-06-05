---
name: krkn-health-check
description: >
  Scaffold a new custom health check plugin for krkn. Use this skill when the user
  wants to add a new health check type to krkn -- for example, "add a gRPC health check",
  "monitor a database endpoint during chaos", or "check Kafka consumer lag". Guides the
  user through naming, implementing the abstract interface, choosing a threading model,
  writing a unit test, and wiring config.yaml. The authoritative reference for this
  plugin system is krkn/krkn/health_checks/README.md -- read it before generating any code.
user_invocable: true
arguments:
  - name: request
    description: >
      Natural language description of what to monitor -- e.g. "gRPC health check",
      "Kafka consumer lag", "PostgreSQL liveness probe". Include any known details
      such as the protocol, config fields needed, or target namespace.
    required: true
---

# Krkn Health Check Plugin Scaffolder

You are a krkn contributor helping a developer add a new health check plugin to the
krkn chaos engineering framework. Your job is to produce a complete, correct, copy-paste-ready
implementation that passes the `HealthCheckFactory`'s strict naming validation and integrates
with the main run loop without any changes to `run_kraken.py`.

The authoritative reference for this system is:
`krkn/krkn/health_checks/README.md`

Read it before generating anything. Every naming rule, threading contract, and config
convention comes from there and from the source files in `krkn/krkn/health_checks/`.

---

## Step 0: Read the Authoritative Sources

Before writing any code, read these files to ground your output in the actual implementation:

```bash
# Plugin system docs and conventions
cat krkn/krkn/health_checks/README.md

# Abstract base class -- required methods and their signatures
cat krkn/krkn/health_checks/abstract_health_check_plugin.py

# Factory -- naming validation logic (snake_to_capital_camel, is_naming_convention_correct)
cat krkn/krkn/health_checks/health_check_factory.py

# Reference implementation -- threading model, telemetry, stop_event loop guard
cat krkn/krkn/health_checks/http_health_check_plugin.py
```

If the user is working from a repo without these files, tell them and stop.

---

## Step 1: Understand the Request

Parse `{{ request }}` to extract:

- **What to monitor**: protocol, service type, endpoint kind (HTTP, gRPC, DB, queue, etc.)
- **Config fields needed**: what parameters the user will need to specify in `config.yaml`
  (e.g. host, port, topic, credentials, interval, threshold)
- **Threading model**: Does the check need to spawn its own internal threads (like
  `VirtHealthCheckPlugin`) or run in a single external thread (like `HttpHealthCheckPlugin`)?
  Default to the external-thread model (`manages_own_threads() -> False`) unless the user
  explicitly needs internal parallelism.
- **Failure definition**: What does "unhealthy" mean for this check? What return value should
  it set? (`3` = health check failure is the standard for non-critical failures)

Note what was explicitly provided and what you will need to ask about in Step 2.

---

## Step 2: Gather Missing Details

Before writing code, compare what the user provided against what is needed to produce a
complete, runnable plugin. Ask for anything that is missing or ambiguous in a single message.

**Always confirm:**
- The short name for this plugin (e.g. `grpc`, `kafka_lag`, `postgres`) -- this drives all
  file and class naming. If the user hasn't implied one, suggest one and confirm.
- The `config_key` -- the top-level `config.yaml` key this plugin will read from. Suggest
  `<name>_checks` and confirm. It must be unique across all existing plugins.
- The minimum config fields needed (at minimum: `interval`; plus whatever the check needs).
- Whether `exit_on_failure` behavior is needed (should a failing check abort the chaos run?).

**Check for config key conflicts:**
```bash
grep -r "get_config_key" krkn/krkn/health_checks/ --include="*.py" -A2
```
If the suggested config key is already taken, flag it and propose an alternative.

**Ask about threading only if unclear.** The default (external thread, `manages_own_threads()
-> False`) is correct for the vast majority of checks. Only ask if the user describes
something that inherently needs internal parallelism (e.g. "check N VMs in parallel").

Wait for the user's answers before proceeding to Step 3.

---

## Step 3: Derive Names

From the confirmed short name (e.g. `grpc`), derive all required identifiers:

| Artifact | Convention | Example |
|---|---|---|
| File name | `<name>_health_check_plugin.py` | `grpc_health_check_plugin.py` |
| Class name | `<Name>HealthCheckPlugin` (CapitalCamelCase) | `GrpcHealthCheckPlugin` |
| Health check type | `<name>_health_check` | `grpc_health_check` |
| Config key | `<name>_checks` (or user-confirmed) | `grpc_checks` |
| Test file | `tests/test_<name>_health_check_plugin.py` | `tests/test_grpc_health_check_plugin.py` |

**Validation rule from the factory**: the CapitalCamelCase class name must exactly equal
`snake_string.title().replace("_", "")` applied to the module file name (without path or `.py`).
Verify this before generating. Example:
- `grpc_health_check_plugin` → `GrpcHealthCheckPlugin` ✓
- `kafka_lag_health_check_plugin` → `KafkaLagHealthCheckPlugin` ✓

If the name conversion would produce an unexpected class name, flag it and confirm with the user.

---

## Step 4: Generate the Plugin

Produce the complete plugin file at `krkn/krkn/health_checks/<file_name>.py`.

Structure it exactly like `http_health_check_plugin.py`:

1. **Apache 2.0 license header** -- copy from any existing plugin.
2. **Module docstring** -- what the plugin monitors, plus a minimal `config.yaml` example.
3. **Imports** -- standard library first, then third-party, then krkn local. Only import what
   is actually used.
4. **Class** extending `AbstractHealthCheckPlugin`.
5. **`__init__`** -- accept `health_check_type`, `iterations`, and `**kwargs`. Call `super().__init__(health_check_type)`. Store `iterations` and initialize `current_iterations = 0`.
6. **`get_health_check_types()`** -- return a list with at least one unique string.
7. **`get_config_key()`** -- return the confirmed config key string.
8. **`increment_iterations()`** -- `self.current_iterations += 1`.
9. **`run_health_check(config, telemetry_queue)`** -- the main loop. Must:
   - Guard with `while self.current_iterations < self.iterations and not self._stop_event.is_set()`
   - Read `interval` from config with a safe default: `config.get("interval", 5)`
   - Call `self.set_return_value(3)` on health check failure (do NOT raise)
   - Sleep `interval` seconds at the end of each loop iteration
   - Put telemetry data into `telemetry_queue` before returning
10. **`manages_own_threads()`** -- omit (inherits `False`) unless the plugin spawns its own threads.

**Do not** add methods, properties, or config fields beyond what the user requested. No
feature flags, no backwards-compatibility shims, no speculative parameters.

---

## Step 5: Generate the Unit Test

Produce the complete test file at `tests/test_<name>_health_check_plugin.py`.

Use `unittest` (not pytest). Structure following `http_health_check_plugin` tests as a model:

```bash
# Check if a test file already exists for reference
ls tests/test_*health_check* 2>/dev/null
cat tests/test_http_health_check_plugin.py 2>/dev/null
```

The test file must cover:

1. **Plugin loads via factory** -- `HealthCheckFactory()` discovers the plugin; the type string
   appears in `factory.loaded_plugins`.
2. **Plugin creation** -- `factory.create_plugin("<type>", iterations=N)` returns an instance
   with correct `iterations` and `current_iterations == 0`.
3. **`increment_iterations`** -- calling it advances `current_iterations` by 1.
4. **`get_config_key`** -- returns the expected string.
5. **`run_health_check` success path** -- mock the external dependency (HTTP client, DB driver,
   etc.), run for 1 iteration, assert `get_return_value() == 0` and telemetry was put in the queue.
6. **`run_health_check` failure path** -- mock a failure response, assert `get_return_value() == 3`.
7. **Stop event** -- set `plugin._stop_event` before starting and verify the loop exits immediately.

Mock all external dependencies with `unittest.mock.MagicMock` or `patch`. Tests must not
make real network calls, open sockets, or require running services.

---

## Step 6: Show the config.yaml Snippet

Show the user exactly what to add to their `config.yaml` to activate the plugin:

```yaml
# Add this section to config.yaml to enable <Name> health checking
<config_key>:
  interval: 5          # seconds between checks
  <field>: <value>     # plugin-specific fields
  exit_on_failure: false  # set to true to abort chaos run on health failure
```

Explain each field briefly. Note which fields are required vs optional with their defaults.

---

## Step 7: Show the File Checklist

After generating the files, show the user this checklist so they can verify nothing was missed:

```
Files to create:
  [ ] krkn/krkn/health_checks/<name>_health_check_plugin.py
  [ ] tests/test_<name>_health_check_plugin.py

Verify before opening a PR:
  [ ] File name ends with `_health_check_plugin.py`
  [ ] Class name ends with `HealthCheckPlugin` and matches file name in CapitalCamelCase
  [ ] `get_health_check_types()` returns at least one unique type string
  [ ] `get_config_key()` returns a key not used by any other plugin
  [ ] Loop guard uses both `current_iterations < iterations` AND `not self._stop_event.is_set()`
  [ ] `set_return_value(3)` called on failure (not raise, not return early)
  [ ] Telemetry put into `telemetry_queue` before returning from `run_health_check`
  [ ] Unit test covers: factory loads it, create_plugin works, success path, failure path, stop event
  [ ] No changes needed to run_kraken.py (factory auto-discovers via config key map)

Run tests:
  python -m unittest tests.test_<name>_health_check_plugin -v

Verify factory loads the plugin:
  python -c "
  from krkn.health_checks import HealthCheckFactory
  f = HealthCheckFactory()
  print('Loaded:', list(f.loaded_plugins.keys()))
  print('Config keys:', f.config_key_map)
  print('Failed:', f.failed_plugins)
  "
```

---

## Important Guidelines

### Naming is enforced at runtime -- silent failures are the risk
The `HealthCheckFactory` validates names with `is_naming_convention_correct()`. A plugin
that violates naming conventions will appear in `factory.failed_plugins` and be silently
skipped -- no exception is raised in the main run loop. Always validate the snake_case to
CapitalCamelCase conversion before generating file and class names.

### Do not modify run_kraken.py
The factory's `start_all()` method auto-discovers plugins via `config_key_map`. A new plugin
that declares a unique `get_config_key()` and has a matching section in `config.yaml` is
picked up automatically. If you find yourself suggesting changes to `run_kraken.py`, stop
and re-read the factory's `start_all()` method.

### Thread safety
`self.ret_value` is written by the plugin thread and read by the main thread after join.
Use `self.set_return_value()` -- do not write `self.ret_value` directly. Do not add other
shared mutable state without synchronization.

### Telemetry
Put a list of telemetry objects (or an empty list `[]`) into `telemetry_queue` before
returning from `run_health_check`. The main loop reads from the queue after the worker joins.
A plugin that never puts to the queue will cause the main loop to block on `queue.get()`.
If the check produces no structured telemetry, put `[]`.

### exit_on_failure
If the user wants `exit_on_failure` support, read `config.get("exit_on_failure", False)`
inside the loop and call `self.set_return_value(3)` only when it is `True` AND the current
`ret_value` is still `0` (to avoid overwriting a previously set failure code).

### manages_own_threads
Only override `manages_own_threads()` to return `True` if the plugin spawns internal threads
and `run_health_check()` returns immediately after spawning them. In that case the plugin
must also expose a `thread_join()` method that blocks until all internal threads complete.
The factory's `start_all()` checks this flag and skips wrapping such plugins in an external
thread. The `VirtHealthCheckPlugin` is the only current example -- read it before using
this pattern.
