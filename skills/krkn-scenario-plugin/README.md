# krkn-scenario-plugin

A Claude Code skill that scaffolds a new chaos scenario plugin for [krkn](https://github.com/krkn-chaos/krkn) -- complete with the plugin directory, implementation, `__init__.py`, unit test, and example scenario YAML.

## Installation

```bash
npx skills add https://github.com/krkn-chaos/krkn-skills --skills krkn-scenario-plugin
```

<details>
<summary>Alternative installation methods</summary>

### Download to a specific project

```bash
mkdir -p .claude/skills
curl -o .claude/skills/krkn-scenario-plugin.md https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-scenario-plugin/SKILL.md
```

### Global installation (available in all projects)

```bash
mkdir -p ~/.claude/skills
curl -o ~/.claude/skills/krkn-scenario-plugin.md https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-scenario-plugin/SKILL.md
```

</details>

## Usage

```
/krkn-scenario-plugin add a DNS disruption scenario for pods
/krkn-scenario-plugin create a custom memory pressure scenario for specific nodes
/krkn-scenario-plugin implement a graceful pod restart with configurable delay
/krkn-scenario-plugin add a new AWS node termination scenario
```

## What it generates

| Artifact | Description |
|---|---|
| `krkn/krkn/scenario_plugins/<name>/` | Plugin directory (name must not contain "scenario" or "plugin") |
| `krkn/krkn/scenario_plugins/<name>/__init__.py` | Required for factory auto-discovery |
| `krkn/krkn/scenario_plugins/<name>/<name>_scenario_plugin.py` | Full plugin extending `AbstractScenarioPlugin` |
| `tests/test_<name>_scenario_plugin.py` | `unittest` test file covering factory discovery, success/failure paths |
| `scenarios/openshift/<name>_scenario.yaml` | Example scenario config with all fields documented |

## How the plugin system works

Plugins are auto-discovered by `ScenarioPluginFactory` via `pkgutil.walk_packages` -- no changes to `run_kraken.py` needed. The factory enforces three strict naming rules that cause silent load failures if violated:

1. File name must end with `_scenario_plugin.py`
2. Directory name must NOT contain the words `scenario` or `plugin`
3. Class name must be the exact CapitalCamelCase of the file name

The skill validates all three before generating any code. A scenario is activated by adding its type string to the `chaos_scenarios` list in `config.yaml`.
