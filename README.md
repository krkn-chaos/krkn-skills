# Krkn Scenario Generator - Claude Code Skill

A Claude Code skill that generates production-ready [Krkn](https://github.com/krkn-chaos/krkn) chaos engineering scenarios. It auto-clones the [krkn-knowledgebase](https://github.com/ddjain/krkn-knowledgebase) and pulls the latest data on every invocation.

## What it does

Type `/krkn-scenario <description>` in any Claude Code session and get:
- Validated `krknctl` CLI commands
- Validated `krkn-hub` Docker commands
- Parameter tables with descriptions
- Relevant edge cases and notes

All generated from the authoritative knowledge base -- no hallucinated flags or env vars.

## Quick Examples

```
/krkn-scenario kill etcd pods in openshift-etcd namespace
/krkn-scenario add 200ms network latency to worker nodes for 5 minutes
/krkn-scenario hog 4 CPU cores at 80% on nodes labeled stress-test=true
/krkn-scenario fill PVCs to 90% in namespace my-app
/krkn-scenario simulate zone outage in us-east-1a
```

## Installation

```bash
npx skills add ddjain/krkn-skill
```

That's it. The skill is now available in your Claude Code sessions. Start using it:

```
/krkn-scenario kill pods in namespace kube-system
```

<details>
<summary>Alternative installation methods</summary>

### Download to a specific project

```bash
mkdir -p .claude/skills
curl -o .claude/skills/krkn-scenario.md https://raw.githubusercontent.com/ddjain/krkn-skill/main/SKILL.md
```

### Global installation (available in all projects)

```bash
mkdir -p ~/.claude/skills
curl -o ~/.claude/skills/krkn-scenario.md https://raw.githubusercontent.com/ddjain/krkn-skill/main/SKILL.md
```

### Clone this repo

```bash
git clone https://github.com/ddjain/krkn-skill.git
```

Then register in your project's `.claude/settings.local.json`:

```json
{
  "skills": {
    "krkn-scenario": {
      "path": "/path/to/krkn-skill/SKILL.md"
    }
  }
}
```

</details>

## How it Works

1. On every invocation, the skill clones/pulls the latest [krkn-knowledgebase](https://github.com/ddjain/krkn-knowledgebase) to `~/.krkn/knowledgebase/`
2. Matches your natural language request to one of 20 scenario types
3. Reads the full scenario definition JSON with all parameters, validators, and mappings
4. Generates validated commands for both `krknctl` (CLI) and `krkn-hub` (Docker)

## Supported Scenarios

| Category | Scenarios |
|----------|-----------|
| **Pod-Level** | pod-scenarios, container-scenarios |
| **Node-Level** | node-scenarios, node-scenarios-bm, node-cpu-hog, node-memory-hog, node-io-hog |
| **Network** | network-chaos, pod-network-chaos, node-network-filter, pod-network-filter, syn-flood |
| **Time** | time-scenarios |
| **Application** | application-outages |
| **Service** | service-disruption-scenarios, service-hijacking |
| **Storage** | pvc-scenarios |
| **Cluster-Wide** | power-outages, zone-outages |
| **Virtualization** | kubevirt-outage |

## Updating

The skill auto-pulls the latest knowledge base on every use. To update the skill itself:

```bash
# If installed via npx skills
npx skills add ddjain/krkn

# If installed via curl
curl -o ~/.claude/skills/krkn-scenario.md https://raw.githubusercontent.com/ddjain/krkn-skill/main/SKILL.md

# If cloned
git -C /path/to/krkn-skill pull
```
