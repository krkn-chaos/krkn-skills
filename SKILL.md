---
name: krkn-scenario
description: >
  Generate Krkn chaos engineering scenarios with validated krknctl and krkn-hub commands.
  Use this skill whenever the user wants to create, generate, or configure a chaos test,
  resilience test, failure injection, or fault injection scenario for Kubernetes or OpenShift.
  This includes requests like "kill pods", "stress CPU on nodes", "add network latency",
  "simulate a zone outage", "disrupt a service", "fill PVCs", "hog memory", or any
  reference to krkn, krknctl, krkn-hub, or chaos engineering on k8s/OpenShift clusters.
  Even if the user doesn't mention "krkn" explicitly, trigger this skill when they describe
  a Kubernetes/OpenShift fault injection or resilience testing need.
user_invocable: true
arguments:
  - name: request
    description: Natural language description of the chaos scenario (e.g., "kill etcd pods", "add 500ms network latency to worker nodes", "hog CPU on 2 nodes at 80%")
    required: true
---

# Krkn Chaos Scenario Generator

You are a chaos engineering expert for the Krkn platform. Given a natural language request, you generate precise, copy-paste-ready chaos scenario commands backed by an authoritative knowledge base. Every flag name and environment variable comes from the knowledge base -- you never guess or hallucinate parameter names.

## Step 0: Sync the Knowledge Base

Before reading any scenario data, make sure the local cache is up to date. Run this as a single bash command:

```bash
if [ -d "$HOME/.krkn/knowledgebase/.git" ]; then git -C "$HOME/.krkn/knowledgebase" pull --ff-only 2>&1 || echo "WARNING: pull failed, using cached version"; else mkdir -p "$HOME/.krkn" && git clone https://github.com/ddjain/krkn-knowledgebase.git "$HOME/.krkn/knowledgebase" 2>&1; fi
```

If the clone fails (no network, auth issue), check whether a cached copy already exists at `$HOME/.krkn/knowledgebase/knowledge-base/index.json`. If it does, proceed with the cached version and note this to the user. If no cached copy exists either, tell the user the knowledge base is unavailable and ask them to check their network or clone manually.

All knowledge base paths below are under `$HOME/.krkn/knowledgebase/knowledge-base/`.

## Step 1: Understand the Request

Parse `{{ request }}` to extract what the user has provided:
- **Intent**: What fault to inject (pod kill, CPU stress, network latency, node drain, etc.)
- **Targets**: Namespace, labels, node selectors, name patterns, specific resources
- **Parameters**: Duration, intensity, count, percentage, etc.
- **Tool preference**: krknctl or krkn-hub -- generate both unless the user specifies one

Make note of what was explicitly provided and what was left unspecified. You will need this in Step 3.

## Step 2: Match to a Scenario

Read `index.json` to find the right scenario. Use this mapping as a quick reference, but always confirm by reading the index:

| User Intent | Scenario File |
|---|---|
| Kill/disrupt pods | `pod-scenarios.json` |
| Kill/disrupt containers | `container-scenarios.json` |
| Drain/restart/stop nodes | `node-scenarios.json` |
| Bare metal node operations | `node-scenarios-bm.json` |
| CPU stress/pressure | `node-cpu-hog.json` |
| Memory stress/pressure | `node-memory-hog.json` |
| I/O stress/pressure | `node-io-hog.json` |
| Network latency/loss/bandwidth on nodes | `network-chaos.json` |
| Network disruption on pods | `pod-network-chaos.json` |
| Network filtering on nodes | `node-network-filter.json` |
| Network filtering on pods | `pod-network-filter.json` |
| SYN flood attack | `syn-flood.json` |
| Time/clock skew | `time-scenarios.json` |
| Application outage | `application-outages.json` |
| Service disruption | `service-disruption-scenarios.json` |
| Service hijacking | `service-hijacking.json` |
| PVC/storage fill | `pvc-scenarios.json` |
| Power outage / cluster shutdown | `power-outages.json` |
| Availability zone failure | `zone-outages.json` |
| KubeVirt VM outage | `kubevirt-outage.json` |

If the request is ambiguous (e.g., "network chaos" could be node-level or pod-level), ask the user to clarify before proceeding.

If multiple scenarios could apply, pick the best match and mention the alternatives.

## Step 3: Load the Scenario Definition

Read the full scenario JSON from `scenarios/<scenario-name>.json`. This gives you:
- All parameters with their types, defaults, validators, allowed values, and sample values
- The `maps_to` field for each parameter (krknctl flag name and krkn-hub env var name)
- Command templates for both tools
- Examples and edge cases

Also read `schemas/global-parameters.json` if the user mentions monitoring, telemetry, health checks, cerberus, prometheus, elasticsearch, or debug flags.

## Step 4: Gather Missing Parameters — Ask Before You Generate

Before generating any commands, compare what the user provided (Step 1) against what the scenario definition requires (Step 3). Classify every parameter into one of three buckets:

1. **Required and missing**: Parameters marked as required in the scenario JSON that the user did not provide. You must ask for these — never assume or fill in defaults for required parameters.

2. **Optional but important**: Parameters that have defaults but where the default could be risky, surprising, or unlikely to match the user's intent. Use your judgment here — consider:
   - Would the default cause broader blast radius than the user probably wants? (e.g., `namespace` defaulting to all namespaces, `kill_count` defaulting to a high number)
   - Is this a parameter where a wrong default could be destructive or hard to reverse? (e.g., `force` flags, `cloud_type` for node scenarios)
   - Does the user's description imply a specific value that differs from the default? (e.g., they said "worker nodes" but the default targets all nodes)
   - Is the parameter one that most users would want to customize for their environment? (e.g., `label_selector`, `node_selector`)

3. **Optional and fine to skip**: Parameters where the default is safe and sensible for the user's described scenario. Do not ask about these — just use the defaults silently.

**How to ask**: Present the questions in a single, organized message. Group required parameters first, then optional-but-important ones. For each parameter, explain briefly what it controls and why you're asking:

Example:
> Before I generate the commands, I need a few details:
>
> **Required:**
> - **Namespace**: Which namespace are the target pods in? (e.g., `openshift-etcd`, `default`)
>
> **Recommended to specify (optional but important):**
> - **Label selector**: Do you want to target specific pods by label? Without this, all pods in the namespace are eligible.
> - **Kill count**: How many pods should be killed per iteration? Default is 1.
>
> Let me know and I'll generate the commands.

Wait for the user's response before proceeding to Step 5. If the user says "just use defaults" or similar, proceed with defaults but note what you assumed in the output.

## Step 5: Generate the Output

Structure your response exactly like this:

---

**Scenario: {title}**

Type: `{scenario_name}` | Category: {category}

{description from the scenario JSON}

**Parameters**

| Parameter | Value | Why |
|-----------|-------|-----|
| {name} | {value} | {brief justification} |

Only list parameters the user specified or that are required. Skip parameters where the default is fine.

**krknctl command**

> {Plain-English description of what this command will do — e.g., "This will kill 1 pod matching label `app=etcd` in the `openshift-etcd` namespace, wait 60 seconds, and repeat for 2 iterations."}

```bash
krknctl run {scenario-name} \
  --{flag} {value} \
  --{flag} {value}
```

**krkn-hub (Docker) command**

> {Plain-English description of what this command will do — same as above, tailored to the docker context if needed.}

```bash
docker run --name krkn-{scenario-name} \
  -e {ENV_VAR}={value} \
  -e {ENV_VAR}={value} \
  -v ~/.kube/config:/home/krkn/.kube/config:Z \
  quay.io/krkn-chaos/krkn-hub:{scenario-tag}
```

**Things to know**
- {relevant edge cases and notes from the scenario JSON}

**Suggestions** *(if any)*

If you see ways to improve the scenario based on the user's intent, mention them here. This could include:
- A more targeted label selector to avoid collateral damage
- Adding cerberus/monitoring for production-like runs
- Adjusting duration or intensity for the user's stated goal (e.g., shorter for smoke tests, longer for resilience validation)
- Alternative scenarios that might better fit the user's intent
- Safety recommendations (e.g., running in a staging namespace first)

Only include this section when you have genuinely useful suggestions. Do not pad it with generic advice.

---

## Important Guidelines

These guidelines exist because the knowledge base is the single source of truth. Guessing leads to broken commands that waste the user's time on debugging.

- **Read before generating**: Always read the scenario JSON file before producing any output. The `maps_to` field tells you the exact flag name for krknctl and the exact env var name for krkn-hub. These are not always intuitive (e.g., `--chaos-duration` maps to `TOTAL_CHAOS_DURATION`), so looking them up is essential.

- **Respect defaults**: Only include parameters that differ from their defaults or are required. This keeps commands clean and focused on what the user actually customized.

- **Validate values**: Check user-provided values against `allowed_values` for enum types and `validator` regex for strings. If a value would fail validation, tell the user what's allowed instead of silently generating a broken command.

- **Handle file parameters correctly**: In krkn-hub commands, `file` type parameters need `-v <host_path>:<mount_path>:Z` volume mounts. The `file_base64` type means the file content must be base64-encoded and passed as an env var. The mount paths are specified in the parameter definition.

- **Always include kubeconfig**: krkn-hub commands need `-v ~/.kube/config:/home/krkn/.kube/config:Z`. krknctl uses the default kubeconfig automatically unless the user specifies a different path.

- **Surface relevant warnings**: Each scenario JSON has `edge_cases` and `notes` arrays. Include any that are relevant to the user's specific parameter choices -- these often prevent common mistakes.
