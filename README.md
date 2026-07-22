# Krkn Claude Code Skills

Claude Code skills for the [krkn-chaos](https://github.com/krkn-chaos) ecosystem -- a CNCF sandbox chaos engineering platform for Kubernetes.

## Available Skills

| Skill | Command | Description |
|-------|---------|-------------|
| [krkn-scenario](skills/krkn-scenario/) | `/krkn-scenario <description>` | Generate validated chaos scenario commands for krknctl and krkn-hub |
| [krkn-pr-review](skills/krkn-pr-review/) | `/krkn-pr-review <pr-ref>` | Review PRs across krkn, krkn-hub, and krknctl with cross-repo analysis |
| [krkn-operator-local](skills/krkn-operator-local/) | `/krkn-operator-local <console-pr> [operator-pr]` | Check out operator PRs, build the backend, and get a local URL to test |

### krkn-scenario

Generates production-ready chaos engineering scenarios backed by the [krkn-knowledgebase](https://github.com/ddjain/krkn-knowledgebase). Produces validated `krknctl` CLI and `krkn-hub` Docker commands.

```
/krkn-scenario kill etcd pods in openshift-etcd namespace
/krkn-scenario add 200ms network latency to worker nodes for 5 minutes
/krkn-scenario hog 4 CPU cores at 80% on nodes labeled stress-test=true
```

### krkn-pr-review

Reviews pull requests with language-specific analysis (Python, Shell/Dockerfile, Go) and cross-repo contract validation. Detects env.sh/krknctl-input.json drift, plugin naming violations, label regex mismatches, and more. **Suggestion-only -- never posts to GitHub.**

```
/krkn-pr-review https://github.com/krkn-chaos/krkn/pull/1297
/krkn-pr-review krkn#1305
/krkn-pr-review krkn-hub#330
/krkn-pr-review krknctl#142
```

### krkn-operator-local

Checks out a `krkn-operator-console` PR (and optionally a `krkn-operator` PR), builds the Go backend, installs frontend dependencies, and prints ready-to-run terminal commands with the local test URL (`http://localhost:3000`). Handles stashing uncommitted changes, fork remotes, dependency checks, and port-conflict warnings automatically.

```
/krkn-operator-local 42
/krkn-operator-local https://github.com/krkn-chaos/krkn-operator-console/pull/42
/krkn-operator-local 42 https://github.com/krkn-chaos/krkn-operator/pull/17
/krkn-operator-local krkn-chaos/krkn-operator-console#42 krkn-chaos/krkn-operator#17
```

## Installation

```bash
npx skills add https://github.com/krkn-chaos/krkn-skills
```

All skills are now available in your Claude Code sessions.

<details>
<summary>Alternative installation methods</summary>

### Download individual skills to a project

```bash
mkdir -p .claude/skills

# Scenario generator
curl -o .claude/skills/krkn-scenario.md \
  https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-scenario/SKILL.md

# PR reviewer
curl -o .claude/skills/krkn-pr-review.md \
  https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-pr-review/SKILL.md


# Operator local test environment
curl -o .claude/skills/krkn-operator-local.md \
  https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-operator-local/SKILL.md
```

### Global installation (available in all projects)

```bash
mkdir -p ~/.claude/skills

curl -o ~/.claude/skills/krkn-scenario.md \
  https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-scenario/SKILL.md

curl -o ~/.claude/skills/krkn-pr-review.md \
  https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-pr-review/SKILL.md

curl -o ~/.claude/skills/krkn-operator-local.md \
  https://raw.githubusercontent.com/krkn-chaos/krkn-skills/main/skills/krkn-operator-local/SKILL.md
```

### Clone this repo

```bash
git clone https://github.com/krkn-chaos/krkn-skills.git
```

Then register in your project's `.claude/settings.local.json`:

```json
{
  "skills": {
    "krkn-scenario": {
      "path": "/path/to/krkn-skills/skills/krkn-scenario/SKILL.md"
    },
    "krkn-pr-review": {
      "path": "/path/to/krkn-skills/skills/krkn-pr-review/SKILL.md"
    },
    "krkn-operator-local": {
      "path": "/path/to/krkn-skills/skills/krkn-operator-local/SKILL.md"
    }
  }
}
```

</details>

## Updating

```bash
# If installed via npx skills
npx skills add https://github.com/krkn-chaos/krkn-skills

# If installed via curl -- re-run the curl commands above

# If cloned
git -C /path/to/krkn-skills pull
```
