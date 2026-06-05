---
name: krkn-scenario-plugin
description: >
  Scaffold a new chaos scenario plugin for krkn. Use this skill when the user wants to add
  a new scenario type to the krkn engine -- for example, "add a DNS disruption scenario",
  "create a custom pod chaos plugin", or "implement a new node stress scenario". Guides the
  user through naming, directory structure, implementing the abstract interface, writing a
  unit test, and adding a scenario YAML example. The authoritative references are
  krkn/CLAUDE.md and krkn/krkn/scenario_plugins/abstract_scenario_plugin.py -- read them
  before generating any code.
user_invocable: true
arguments:
  - name: request
    description: >
      Natural language description of the scenario -- e.g. "DNS disruption for pods",
      "custom memory pressure on specific nodes", "graceful pod restart with configurable
      delay". Include any known details such as target resources, config parameters, or
      cloud provider requirements.
    required: true
---

# Krkn Scenario Plugin Scaffolder

You are a krkn contributor helping a developer add a new chaos scenario plugin to the
krkn chaos engineering framework. Your job is to produce a complete, correct, copy-paste-ready
implementation that passes the `ScenarioPluginFactory`'s strict naming validation and
integrates with the main run loop without any changes to `run_kraken.py`.

The authoritative references for this system are:
- `krkn/CLAUDE.md` (plugin architecture section)
- `krkn/krkn/scenario_plugins/abstract_scenario_plugin.py`
- `krkn/krkn/scenario_plugins/scenario_plugin_factory.py`

Read them before generating anything. Naming violations cause silent factory load failures
at runtime.

---

## Step 0: Read the Authoritative Sources

Before writing any code, read these files:

```bash
# Plugin architecture docs and conventions
grep -A 40 "Plugin Architecture" krkn/CLAUDE.md

# Abstract base class -- required methods and their signatures
cat krkn/krkn/scenario_plugins/abstract_scenario_plugin.py

# Factory -- naming validation logic (is_naming_convention_correct, directory rules)
cat krkn/krkn/scenario_plugins/scenario_plugin_factory.py

# Reference implementation -- run() structure, yaml parsing, telemetry, return codes
cat krkn/krkn/scenario_plugins/pod_disruption/pod_disruption_scenario_plugin.py
```

If the user is working from a repo without these files, tell them and stop.

---

## Step 1: Understand the Request

Parse `{{ request }}` to extract:

- **What fault to inject**: the chaos action (pod kill, network disruption, DNS failure,
  memory pressure, etc.)
- **Target resources**: pods, nodes, namespaces, labels, cloud provider, etc.
- **Config parameters**: what fields the user needs in the scenario YAML (duration,
  count, label selectors, etc.)
- **Cloud provider requirements**: does the scenario need cloud API access (AWS, Azure,
  GCP, etc.)? If so, note which provider.
- **Return value semantics**: what counts as success (0), scenario failure (1), or
  critical alert (2)?

Note what was explicitly provided and what you will need to ask about in Step 2.

---

## Step 2: Gather Missing Details

Before writing code, ask for anything missing in a single message.

**Always confirm:**
- The short name for this plugin (e.g. `dns_disruption`, `memory_pressure`) -- this drives
  all directory and file naming. Suggest one from the request and confirm.
- The scenario type string(s) -- what the user will put under `scenario_type:` in their
  scenario YAML. Suggest `<name>_scenarios` and confirm. Must be unique across all plugins.
- The minimum scenario YAML config fields needed.
- Whether this is cloud-provider-specific (if so, which provider).

**Check for scenario type conflicts:**
```bash
grep -r "get_scenario_types" krkn/krkn/scenario_plugins/ --include="*.py" -A3
```
If the suggested type string is already taken, flag it and propose an alternative.

Wait for the user's answers before proceeding to Step 3.

---

## Step 3: Derive Names

From the confirmed short name (e.g. `dns_disruption`), derive all required identifiers:

| Artifact | Convention | Example |
|---|---|---|
| Directory | `krkn/scenario_plugins/<name>/` | `krkn/scenario_plugins/dns_disruption/` |
| File name | `<name>_scenario_plugin.py` | `dns_disruption_scenario_plugin.py` |
| Class name | `<Name>ScenarioPlugin` (CapitalCamelCase) | `DnsDisruptionScenarioPlugin` |
| Scenario type | `<name>_scenarios` (or user-confirmed) | `dns_disruption_scenarios` |
| Test file | `tests/test_<name>_scenario_plugin.py` | `tests/test_dns_disruption_scenario_plugin.py` |
| Scenario YAML | `scenarios/<platform>/<name>_scenario.yaml` | `scenarios/openshift/dns_disruption_scenario.yaml` |

**Three naming rules enforced by the factory -- all cause silent load failures if violated:**

1. **File name** must end with `_scenario_plugin.py`
2. **Directory name** must NOT contain the words `scenario` or `plugin`
   - `dns_disruption/` ✓ — `dns_disruption_scenario/` ✗ — `dns_plugin/` ✗
3. **Class name** must be the CapitalCamelCase equivalent of the file name:
   `snake_string.title().replace("_", "")`
   - `dns_disruption_scenario_plugin` → `DnsDisruptionScenarioPlugin` ✓

Verify all three before generating. If the user's chosen name would produce a directory
name containing "scenario" or "plugin", flag it and propose an alternative.

---

## Step 4: Generate the Plugin

Produce the complete plugin file at
`krkn/krkn/scenario_plugins/<dir_name>/<file_name>.py`.

Structure it exactly like `pod_disruption_scenario_plugin.py`:

1. **Apache 2.0 license header** -- copy from any existing plugin.
2. **Imports** -- standard library first, then third-party, then krkn local.
   Always import `yaml`, `logging`, and `AbstractScenarioPlugin`. Only import what is used.
3. **Class** extending `AbstractScenarioPlugin`.
4. **`get_scenario_types()`** -- return a list with at least one unique type string.
5. **`run(run_uuid, scenario, lib_telemetry, scenario_telemetry)`** -- the entry point. Must:
   - Wrap all logic in a `try/except Exception` -- **no exception may propagate out of `run()`**
   - Open and parse the scenario YAML: `with open(scenario, "r") as f: config = yaml.safe_load(f)`
   - Execute the chaos logic
   - Return `0` on success, `1` on scenario failure, `2` for critical alerts
   - Log clearly at each stage with `logging.info` / `logging.error`
6. **`__init__`** -- only needed if the plugin requires extra initialization beyond the base
   class. If adding `__init__`, always call `super().__init__(scenario_type)` first.

**Do not** add methods or config fields beyond what the user requested. The base class
`run_scenarios()` handles iteration, telemetry timestamps, wait duration, and rollback --
do not reimplement these.

### Rollback support
The base class provides `self.rollback_handler` automatically. If the scenario modifies
cluster state that could leave the cluster broken on failure (e.g. cordons a node, deletes
a resource), use the rollback handler to register cleanup actions. Read
`krkn/krkn/rollback/handler.py` for the API. For simple scenarios that are inherently
idempotent or self-cleaning, no rollback logic is needed.

---

## Step 5: Generate the `__init__.py`

Create a minimal `krkn/krkn/scenario_plugins/<dir_name>/__init__.py`:

```python
# Copyright <year> The Krkn Authors
#
# Licensed under the Apache License, Version 2.0 (the "License");
# ...
```

The file can be empty beyond the license header. It is required for Python package
discovery -- the factory uses `pkgutil.walk_packages` to find plugins.

---

## Step 6: Generate the Unit Test

Produce the complete test file at `tests/test_<name>_scenario_plugin.py`.

Use `unittest` (not pytest). Check for existing reference tests:

```bash
ls tests/test_*scenario_plugin* 2>/dev/null | head -5
# Read the simplest one as a model
```

The test file must cover:

1. **Plugin loads via factory** -- `ScenarioPluginFactory()` discovers the plugin; the
   scenario type string appears in `factory.loaded_plugins`.
2. **`get_scenario_types()`** -- returns the expected list.
3. **`run()` success path** -- mock `lib_telemetry`, write a minimal scenario YAML to a
   `tempfile`, call `run()`, assert return value is `0`.
4. **`run()` failure path** -- mock a failure condition, assert return value is `1`.
5. **Missing scenario file** -- pass a path that does not exist, assert return value is `1`
   (or whatever the plugin does for missing config).

Mock all external dependencies (`KrknTelemetryOpenshift`, Kubernetes API, cloud SDKs) with
`unittest.mock.MagicMock` or `patch`. Tests must not require a running cluster.

---

## Step 7: Generate the Scenario YAML

Produce a minimal example scenario config at
`scenarios/openshift/<name>_scenario.yaml` (or `scenarios/kube/` if cluster-agnostic):

```yaml
- config:
    scenario_type: <scenario_type>
    # Plugin-specific fields with sensible defaults
    <field>: <value>
    <field>: <value>
```

Include a comment for each field explaining what it controls. This file is both a test
fixture and the user-facing documentation for how to configure the scenario.

---

## Step 8: Show the config.yaml Wiring

Show the user how to add the scenario to their `config.yaml` chaos run:

```yaml
kraken:
  chaos_scenarios:
    - <name>_scenarios:
        - scenarios/openshift/<name>_scenario.yaml
```

Explain that `<name>_scenarios` must match what `get_scenario_types()` returns.

---

## Step 9: Show the File Checklist

After generating the files, show the user this checklist:

```
Files to create:
  [ ] krkn/krkn/scenario_plugins/<name>/
  [ ] krkn/krkn/scenario_plugins/<name>/__init__.py
  [ ] krkn/krkn/scenario_plugins/<name>/<name>_scenario_plugin.py
  [ ] tests/test_<name>_scenario_plugin.py
  [ ] scenarios/openshift/<name>_scenario.yaml  (or scenarios/kube/)

Verify before opening a PR:
  [ ] Directory name does NOT contain "scenario" or "plugin"
  [ ] File name ends with `_scenario_plugin.py`
  [ ] Class name matches file name via CapitalCamelCase conversion
  [ ] Class extends AbstractScenarioPlugin
  [ ] `run()` is wrapped in try/except -- no exception propagates out
  [ ] `run()` returns 0 (success), 1 (failure), or 2 (critical alert)
  [ ] `get_scenario_types()` returns at least one unique type string
  [ ] __init__.py exists in the plugin directory
  [ ] Unit test covers: factory loads it, success path, failure path
  [ ] Scenario YAML added under scenarios/
  [ ] No changes needed to run_kraken.py

Run tests:
  python -m unittest tests.test_<name>_scenario_plugin -v

Verify factory loads the plugin:
  python -c "
  from krkn.scenario_plugins.scenario_plugin_factory import ScenarioPluginFactory
  f = ScenarioPluginFactory()
  print('Loaded:', list(f.loaded_plugins.keys()))
  print('Failed:', f.failed_plugins)
  "
```

---

## Important Guidelines

### Naming violations cause silent runtime failures
The `ScenarioPluginFactory` validates all three naming rules in `is_naming_convention_correct()`.
A violation means the plugin appears in `factory.failed_plugins` and is silently skipped --
the chaos run proceeds without it and no exception is raised. Always verify:
1. Directory does not contain "scenario" or "plugin"
2. File ends with `_scenario_plugin.py`
3. Class name is the exact CapitalCamelCase of the file name

### Never let exceptions escape run()
The base class `run_scenarios()` wraps `self.run()` in a try/except, but only as a safety
net -- it logs a generic error and sets `return_value = 1`. If you let specific errors
propagate, the caller loses all context. Catch exceptions inside `run()`, log them with
detail, and return the appropriate exit code.

### Do not modify run_kraken.py
The `ScenarioPluginFactory` auto-discovers plugins via `pkgutil.walk_packages`. A new plugin
directory with a correct `__init__.py` and correctly named module is picked up automatically.
If you find yourself suggesting changes to `run_kraken.py`, stop and re-read the factory.

### Return code semantics
- `0` -- scenario completed successfully
- `1` -- scenario failed (recoverable; chaos run continues with remaining scenarios)
- `2` -- critical alert fired (run stops, cluster may be unhealthy)

Only return `2` if you detect a condition that makes continued chaos testing unsafe
(e.g. cluster health check failed, control plane unreachable).

### Cloud-provider-specific plugins
If the scenario requires cloud API access, look at an existing cloud node scenario as a
model (e.g. `krkn/scenario_plugins/node_actions/aws_node_scenarios.py`). Use the same
SDK initialization pattern and error handling. Add any new cloud SDK to `requirements.txt`
with a pinned version and verify it does not conflict with `docker<7.0` or `requests<2.32`.

### Telemetry
`scenario_telemetry` is managed by `run_scenarios()` in the base class -- it sets
`start_timestamp`, `end_timestamp`, and `exit_status` automatically. Inside `run()`, only
write to `scenario_telemetry` fields that are specific to your scenario (e.g.
`scenario_telemetry.affected_pods`). Do not set `exit_status` yourself.
