# krkn-health-check

A Claude Code skill that scaffolds a new custom health check plugin for [krkn](https://github.com/krkn-chaos/krkn) -- complete with the plugin file, unit test, and `config.yaml` snippet.

## Installation

```bash
npx skills add https://github.com/krkn-chaos/krkn-skills --skills krkn-health-check
```

<details>
<summary>Alternative installation methods</summary>

### Download to a specific project

```bash
mkdir -p .claude/skills
curl -o .claude/skills/krkn-health-check.md https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-health-check/SKILL.md
```

### Global installation (available in all projects)

```bash
mkdir -p ~/.claude/skills
curl -o ~/.claude/skills/krkn-health-check.md https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-health-check/SKILL.md
```

</details>

## Usage

```
/krkn-health-check add a gRPC health check for my service
/krkn-health-check monitor Kafka consumer lag during chaos
/krkn-health-check check PostgreSQL liveness on port 5432
/krkn-health-check HTTP health check with bearer token auth and exit_on_failure
```

## What it generates

| Artifact | Description |
|---|---|
| `krkn/krkn/health_checks/<name>_health_check_plugin.py` | Full plugin implementation extending `AbstractHealthCheckPlugin` |
| `tests/test_<name>_health_check_plugin.py` | `unittest` test file covering factory discovery, success/failure paths, and stop event |
| `config.yaml` snippet | Ready-to-paste config section with all fields explained |

## How the plugin system works

Plugins are auto-discovered by `HealthCheckFactory` -- no changes to `run_kraken.py` needed. The factory scans `krkn/health_checks/` for files ending in `_health_check_plugin.py`, validates naming conventions, and maps each plugin's `get_config_key()` to the corresponding `config.yaml` section. A plugin is activated simply by adding its config key to `config.yaml`.

The skill enforces the factory's strict naming rules before generating any code, so the plugin loads correctly on the first try.
