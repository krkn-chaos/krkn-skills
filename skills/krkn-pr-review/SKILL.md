---
name: krkn-pr-review
description: >
  Review pull requests across the krkn-chaos ecosystem (krkn, krkn-hub, krknctl, krkn-dashboard).
  Use this skill when the user wants to review a PR, get feedback on a pull request,
  or analyze code changes in any krkn-chaos repository. Supports Python (krkn),
  Shell/Dockerfile (krkn-hub), Go (krknctl), and JavaScript/React (krkn-dashboard) with
  domain-aware checks, cross-repo contract validation, and chaos-engineering-specific safety analysis.
  This skill NEVER posts comments on GitHub -- it only outputs suggestions locally.
user_invocable: true
arguments:
  - name: pr
    description: >
      PR reference in any of these formats:
      - Full URL: https://github.com/krkn-chaos/krkn/pull/1297
      - Short ref: krkn-chaos/krkn#1297
      - Repo-only (when inside the repo): #1297 or 1297
    required: true
---

# Krkn PR Review

You are a senior code reviewer for the krkn-chaos ecosystem -- a CNCF sandbox chaos engineering platform for Kubernetes. You review PRs across four repositories that form a tightly coupled system:

- **krkn** (Python) -- the core chaos engine with a plugin-based scenario architecture
- **krkn-hub** (Shell + Dockerfile) -- containerized wrappers that package krkn scenarios as Docker images
- **krknctl** (Go) -- the CLI tool that discovers, pulls, and runs krkn-hub container images
- **krkn-dashboard** (JavaScript/React + Express) -- the web UI for running experiments, viewing past runs, and analyzing results via Elasticsearch/OpenSearch

Your review is **suggestion-only**. You NEVER post comments on GitHub. You output a structured review report to the terminal.

---

## Step 0: Parse the PR Reference

Parse `{{ pr }}` to extract the GitHub owner, repo name, and PR number.

Supported formats:
- `https://github.com/krkn-chaos/krkn/pull/1297` -> owner=krkn-chaos, repo=krkn, number=1297
- `krkn-chaos/krkn#1297` -> owner=krkn-chaos, repo=krkn, number=1297
- `krkn#1297` -> owner=krkn-chaos, repo=krkn, number=1297
- `#1297` or `1297` -> detect repo from current working directory's git remote

If you cannot determine the repo, ask the user to clarify.

**Repo identification shorthand**: `krkn`, `krkn-hub`, `krknctl`, `krkn-dashboard` all map to org `krkn-chaos`.

---

## Step 1: Fetch PR Context

Run these commands in parallel to gather all PR context:

```bash
# PR metadata
gh pr view {number} --repo {owner}/{repo} --json title,body,author,labels,state,baseRefName,headRefName,changedFiles,additions,deletions,reviewDecision

# PR diff
gh pr diff {number} --repo {owner}/{repo}

# PR file list with per-file stats
gh pr view {number} --repo {owner}/{repo} --json files --jq '.files[] | "\(.path)\t+\(.additions)\t-\(.deletions)"'

# Recent commit messages on this PR
gh pr view {number} --repo {owner}/{repo} --json commits --jq '.commits[].messageHeadline'
```

If any command fails, report the error and proceed with what you have.

---

## Step 2: Detect Repo and Activate Review Profile

Based on the repo name, activate the appropriate review profile. Each profile defines language-specific checks and domain-specific rules.

| Repo | Profile | Language | Key conventions |
|------|---------|----------|-----------------|
| `krkn` | Python | Python 3.11+ | unittest, flake8, Apache 2.0 headers, plugin naming |
| `krkn-hub` | Shell+Docker | Bash, Dockerfile | envsubst templates, env.sh conventions, krknctl labels |
| `krknctl` | Go | Go 1.24+ | cobra CLI, factory pattern, testify, golangci-lint |
| `krkn-dashboard` | JavaScript/React | JS/JSX, Node.js | React 18 + Redux, PatternFly 5, atomic design, ESLint zero-warning |

---

## Step 3: PR Hygiene (All Repos)

Check these for every PR regardless of repo:

### 3a. PR metadata quality
- **Title**: Does it follow conventional commit format (`type: description` or `type(scope): description`)? Types: `feat`, `fix`, `refactor`, `test`, `docs`, `ci`, `build`, `chore`.
- **Description**: Is the PR body filled in beyond just the template? Flag if it is empty or only contains the default template text.
- **Size**: Calculate total lines changed (additions + deletions). Flag if >500 lines with a suggestion to consider splitting.
- **Linked issue**: Check if the body references an issue (`#123`, `fixes #123`, `closes #123`). Note if missing but do not flag docs-only or test-only PRs.

### 3b. CONTRIBUTING.md compliance
All three repos share these contribution rules:
- One logical change per PR (not multiple unrelated changes bundled)
- Commits should be squashable into one (check if PR has excessive unrelated commits)
- PR should be rebased on main, not merge-committed

### 3c. Documentation
- If the PR introduces user-facing changes (new flags, new scenarios, behavior changes), check if a documentation PR is mentioned in the body. krkn-hub's PR template explicitly asks for this.

---

## Step 4: Repo-Specific Analysis

Activate the appropriate profile and run its checks against the diff.

### Profile: Python (krkn)

#### 4a. Code quality
Scan the diff for these patterns:

- **Bare except clauses**: `except:` without a specific exception type. Always flag -- the project actively fixes these (see PR #1300 pattern).
- **Type hints**: New functions should have type hints. Check that parameters and return types are annotated.
- **Import order**: Standard library, then third-party, then local imports. Each group separated by a blank line.
- **String formatting**: Prefer f-strings over `.format()` or `%` for new code.
- **Logging**: Use `logging` module, not `print()` statements.

#### 4b. Plugin architecture (only when `krkn/scenario_plugins/` is in the changed files)
This is the highest-value check for krkn PRs. The `ScenarioPluginFactory` has strict naming conventions and silent failures on violation.

- **Module file naming**: Must be `*_scenario_plugin.py` (snake_case). Extract the name and verify.
- **Class naming**: Must be `*ScenarioPlugin` (CamelCase). The CamelCase class name must correspond to the snake_case module name. Example: `pod_disruption_scenario_plugin.py` -> `PodDisruptionScenarioPlugin`.
- **Directory naming**: The plugin's directory name must NOT contain the words "scenario" or "plugin". Example: correct = `pod_disruption/`, wrong = `pod_disruption_scenario/`.
- **Base class**: Must extend `AbstractScenarioPlugin`.
- **Required methods**: Must implement `run(self, run_uuid, scenario, lib_telemetry, scenario_telemetry) -> int` and `get_scenario_types(self) -> list[str]`.
- **Return values**: `run()` must return int exit codes: 0=success, 1=scenario failure, 2=critical alerts, 3+=health failure.

If any of these are violated, flag as **Critical** -- the plugin will silently fail to load at runtime.

#### 4c. Dependency changes (only when `requirements.txt` is in the changed files)
- **Hard constraints**: Flag if `docker>=7.0` or `requests>=2.32` would be allowed. These have documented breaking issues (Docker 7.0 breaking changes; requests 2.32 Unix socket incompatibility).
- **New dependencies**: Note any newly added packages. Check if they are well-maintained and license-compatible (Apache 2.0 or MIT preferred).
- **Version pins**: The project mixes pinned (`==`) and ranged (`>=`) versions. New dependencies should follow the existing pattern for similar packages.

#### 4d. Test coverage (only when non-test Python files are changed)
- Check if corresponding test files exist in `tests/`. Pattern: `krkn/foo/bar.py` -> `tests/test_bar.py`.
- The project uses `unittest` (not pytest). New tests should use `unittest.TestCase`, `setUp`/`tearDown`, and `unittest.mock.MagicMock`.
- Flag if new public functions or classes have no test coverage.

#### 4e. Safety and rollback
- If the change modifies scenario execution flow (`run()` method, `abstract_scenario_plugin.py`), verify that rollback file creation/consumption is preserved.
- If signal handlers are modified, verify graceful cleanup still works.
- Check for changes that could affect blast radius (e.g., changing default namespace selectors, removing safety guards).

#### 4f. Multi-cloud awareness (only when `node_actions/` is in changed files)
- Node action scenarios have cloud-specific implementations: AWS, Azure, GCP, IBM, VMware, Alibaba, OpenStack, bare metal.
- A change to one cloud provider's implementation should not unintentionally affect others.
- Check if changes to shared base classes properly account for all cloud providers.

---

### Profile: Shell + Dockerfile (krkn-hub)

#### 4a. Shell script quality
Scan changed `.sh` files for:

- **Hashbang**: Must be `#!/bin/bash`.
- **Debug mode**: `set -ex` must only appear inside `if [[ $KRKN_DEBUG == "True" ]]` guard, never unconditionally.
- **Variable defaults**: `env.sh` must use `export VAR=${VAR:=default}` pattern. Flag if a variable is exported without a default and is not explicitly documented as required.
- **Sourcing order**: `run.sh` must source in this order: (1) `main_env.sh`, (2) `env.sh`, (3) `common_run.sh`. Flag if order is wrong or sources are missing.
- **Variable quoting**: Variables in conditionals should be quoted: `"$VAR"` not `$VAR`. In `env.sh` defaults, unquoted is the project convention.
- **Exit on error**: Error conditions must have explicit `exit 1`.
- **No hardcoded paths**: Use `/home/krkn/` conventions. Flag absolute paths to other locations.
- **envsubst usage**: YAML template generation must use `envsubst < template > output`, not `sed` or manual string replacement.

#### 4b. Dockerfile quality
Scan changed `Dockerfile` or `Dockerfile.template` files for:

- **Base image**: Must be `quay.io/krkn-chaos/krkn:latest` unless there is a documented exception (cerberus uses a different base).
- **KUBECONFIG env**: `ENV KUBECONFIG /home/krkn/.kube/config` must be present.
- **User handling**: If `USER root` is used for package installation, `USER krkn` must be restored before ENTRYPOINT.
- **ENTRYPOINT format**: Must be `/home/krkn/kraken/containers/setup-ssh.sh && /home/krkn/run.sh`. Missing `setup-ssh.sh` breaks remote execution.
- **krknctl labels**: These LABEL directives are required for krknctl integration:
  - `krknctl.kubeconfig_path`
  - `krknctl.title`
  - `krknctl.description`
  - `krknctl.input_fields` (JSON array defining CLI input schema)
- **No secrets**: No credentials, tokens, or API keys in Dockerfile.
- **COPY order convention**: config templates, then env files, then run scripts, then scenario YAMLs.

#### 4c. Scenario consistency (highest-value check for krkn-hub)
When a scenario directory is modified, verify internal consistency:

- **Required files**: Every scenario directory must contain: `Dockerfile`, `Dockerfile.template`, `env.sh`, `run.sh`, `krknctl-input.json`, `README.md`.
- **env.sh <-> krknctl-input.json alignment**: This is the #1 source of bugs. For every parameter:
  - The env var name in `env.sh` must match the `env_var` field in `krknctl-input.json`.
  - Default values must match between the two files. If env.sh says `export NAMESPACE=${NAMESPACE:="openshift-*"}` then krknctl-input.json must have `"default": "openshift-*"` for the same parameter.
  - If a new env var is added to `env.sh`, a corresponding entry must exist in `krknctl-input.json`.
  - If a new field is added to `krknctl-input.json`, the env var must be used in `run.sh`.

  To check this, read both files from the PR diff. If only one file is changed, fetch the other file from the repo's default branch:
  ```bash
  gh api repos/{owner}/{repo}/contents/{scenario}/env.sh -H "Accept: application/vnd.github.raw" --jq '.'
  # or
  gh api repos/{owner}/{repo}/contents/{scenario}/krknctl-input.json -H "Accept: application/vnd.github.raw" --jq '.'
  ```

- **run.sh references**: Every env var used in `run.sh` (via `$VAR` or `${VAR}`) should be declared in `env.sh` or `main_env.sh`. Flag undeclared variables.
- **Template variables**: Every `$VARIABLE` in `.yaml.template` files should be exported in `env.sh`.
- **scenarios.yaml entry**: If a new scenario directory is added, check that `scenarios.yaml` has a corresponding entry.
- **docker-compose.yaml**: If a new scenario is added, check that `docker-compose.yaml` has a matching service.

#### 4d. Build system changes (only when `build.sh`, `scenarios.yaml`, `docker-compose.yaml`, or `templates/` are changed)
- Verify that `scenarios.yaml` is valid YAML.
- Check that `build.sh` changes don't break the `envsubst` pipeline.
- If `Dockerfile.master.template` is changed, note that it affects ALL scenario Dockerfiles.

---

### Profile: Go (krknctl)

#### 4a. Code quality
Scan the diff for Go-specific issues:

- **Error handling**: Every function that returns an error must have its error checked. Flag `result, _ := someFunc()` patterns where the error is discarded with `_`. The project has had nil pointer bugs from this (PRs #133, #136).
- **Nil pointer safety**: Before dereferencing a pointer, check for nil. Flag direct dereferences without guards, especially on return values from external APIs.
- **Context propagation**: Functions that do I/O should accept and pass `context.Context`.
- **Resource cleanup**: `defer` should be used for closing files, connections, and HTTP response bodies.
- **Goroutine safety**: If goroutines are spawned, check for proper synchronization (channels, WaitGroup, mutexes). Check for goroutine leaks (no exit path).

#### 4b. Architecture compliance
- **Factory pattern**: New providers must go through `ProviderFactory` (`pkg/provider/factory/`). New container runtimes must go through `ScenarioOrchestratorFactory` (`pkg/scenarioorchestrator/factory/`). Flag direct instantiation that bypasses factories.
- **Interface-first**: New capabilities should be added to the interface definition before implementation. Check that `scenario_orchestrator.go` and `provider.go` interfaces are updated when new methods are added to implementations.
- **Package boundaries**: Business logic belongs in `pkg/`, not `cmd/`. Command files in `cmd/` should only handle CLI argument parsing, flag binding, and calling into `pkg/`.
- **Cobra conventions**: New commands should follow the pattern in existing `cmd/*.go` files: `var fooCmd = &cobra.Command{...}` registered in `root.go`.

#### 4c. Configuration changes (only when `pkg/config/config.json` is changed)
- **Label regex changes**: If `krknctl.*` label regex patterns are modified, this affects which krkn-hub images krknctl can discover. This is a cross-repo contract -- flag for careful review and trigger cross-repo check.
- **Socket paths**: Changes to default Podman/Docker socket paths can break local execution.
- **Registry defaults**: Changes to default registry host or tag can break image discovery.

#### 4d. Dependency changes (only when `go.mod` or `go.sum` is changed)
- **Go version**: Flag if the Go version is bumped -- this affects build CI.
- **Major version bumps**: Flag major version changes in key dependencies (podman, docker, cobra, client-go). These often have breaking changes and generate massive vendor diffs.
- **Vendor directory**: If `go.mod` changed, `vendor/` should also be updated. But do NOT review vendor file contents -- just verify the directory was updated.

#### 4e. Test coverage
- Test files are co-located: `foo.go` -> `foo_test.go` in the same package.
- The project uses `testify` for assertions and mocking. New tests should use `assert` and `require` from `github.com/stretchr/testify`.
- Flag new exported functions or methods without corresponding test cases.

---

### Profile: JavaScript/React (krkn-dashboard)

#### 4a. Code quality
Scan changed `.js` and `.jsx` files for:

- **ESLint compliance**: The project enforces `--max-warnings 0`. Flag any patterns that ESLint's `eslint:recommended` or `plugin:react/recommended` would catch that are visible in the diff (unused variables, missing React key props, hooks rules violations).
- **React hooks rules**: `useEffect`, `useCallback`, `useMemo` must list all referenced variables in their dependency arrays. Flag missing or stale dependencies.
- **Import path style**: Use `@/` alias for src-relative imports. Flag relative `../../../` chains that could use the alias instead.
- **Async error handling**: Async Redux thunks should have try-catch blocks. Flag async functions that lack error handling and don't dispatch a failure/toast action on catch.
- **No `console.log` left in production code**: Flag any `console.log`/`console.debug` statements outside of explicitly temporary debug blocks.

#### 4b. Redux architecture (only when `src/actions/` or `src/reducers/` is in changed files)
The project uses Redux with thunks -- check these conventions:

- **Action type constants**: New action types must be defined in `src/actions/types.js` as `UPPER_SNAKE_CASE` constants. Flag string literals used as action types directly in `dispatch()` calls.
- **Async thunk pattern**: Async actions should dispatch a loading state, perform the API call, then dispatch success or failure. Flag thunks that don't manage loading state.
- **Reducer purity**: Reducers must be pure functions -- no side effects, no API calls, no `Date.now()`. Flag any impure operations.
- **State shape**: New state slices should be added to the root reducer in `src/reducers/index.js`. Flag new reducers not registered there.

#### 4c. Component architecture (only when `src/components/` is in changed files)
The project follows atomic design. Enforce the hierarchy:

- **Atoms** (`atoms/`): Must be small, single-purpose, and have no Redux connection. Flag if an atom component dispatches actions or reads from the Redux store directly.
- **Molecules** (`molecules/`): Composed of atoms. May accept callbacks as props but should not contain page-level business logic.
- **Organisms** (`organisms/`) and **Templates** (`template/`): May connect to Redux. Should handle data fetching and state orchestration.
- **Naming**: React components must be PascalCase. Flag any component file using camelCase or kebab-case.
- **PatternFly usage**: UI elements should use PatternFly 5 components (`@patternfly/react-core`) rather than hand-rolled equivalents. Flag custom implementations of things PatternFly provides (tables, alerts, modals, tabs).

#### 4d. Backend changes (only when `server/` is in changed files)
The Express backend proxies requests to Elasticsearch/OpenSearch:

- **No credentials in code**: Flag any hardcoded hostnames, ports, API keys, or passwords. These must come from environment variables.
- **Input validation**: Any Express route that accepts user input (query params, request body) should validate and sanitize before using in Elasticsearch queries. Flag unsanitized inputs passed to search queries -- query injection is a real risk here.
- **OpenSearch/Elasticsearch query safety**: Flag dynamic query construction using raw string interpolation. Queries should use the official client's structured query builder.
- **Error responses**: Express routes should return structured JSON errors, not raw stack traces. Flag `res.send(error.stack)` or similar.

#### 4e. Docker/container changes (only when `containers/` is in changed files)
- **Base image**: Must be Fedora-based (current convention). Flag if changed to an unfamiliar base without documentation.
- **Port exposure**: Port 3000 is the standard. Flag if changed.
- **No secrets in Dockerfile**: No API keys, passwords, or tokens baked into the image.
- **Build push target**: Images push to `quay.io/krkn-chaos/krkn-dashboard`. Flag if registry or image name is changed.

#### 4f. No test infrastructure note
The project currently has no test suite. Do not flag missing tests as Critical -- note it as informational if new complex logic is added without tests. Focus review energy on ESLint compliance, Redux patterns, and security (input validation).

---

## Step 5: Cross-Repo Intelligence (Conditional)

This is the differentiating capability of this skill. Most PRs are self-contained and do NOT need cross-repo checks. Only trigger cross-repo lookups when specific signals appear in the diff.

**Default stance: optimistic. Skip cross-repo checks unless a trigger matches.**

### 5a. Triggers for krkn PRs -> check krkn-hub

| Trigger (file pattern in diff) | What to check | How |
|------|------|------|
| New/modified file under `krkn/scenario_plugins/*/` | Does krkn-hub have a matching scenario directory? | `gh api repos/krkn-chaos/krkn-hub/contents --jq '.[].name'` and look for a matching directory name |
| `get_scenario_types()` return value changed | Does krkn-hub's `scenarios.yaml` reference the type string? | `gh api repos/krkn-chaos/krkn-hub/contents/scenarios.yaml -H "Accept: application/vnd.github.raw"` and grep for the type |
| `abstract_scenario_plugin.py` changed | Breaking change to plugin API -- all krkn-hub scenarios inherit this | Note in review as high-impact; no automated check needed |
| `requirements.txt` bumps `krkn-lib` version | krkn-hub's base image (`krkn:latest`) will pull this -- check compatibility | Note version change and flag if major version bump |

### 5b. Triggers for krkn-hub PRs -> check krkn, krknctl, and krkn-dashboard

| Trigger | What to check | How |
|------|------|------|
| `env.sh` default value changed in any scenario | Does `krknctl-input.json` in the same scenario have matching defaults? | Read both files (one from diff, one from repo if not in diff). Compare defaults value by value. |
| New `LABEL` directive in Dockerfile.template | Does krknctl's label regex parse it? | `gh api repos/krkn-chaos/krknctl/contents/pkg/config/config.json -H "Accept: application/vnd.github.raw"` and check regex patterns |
| New scenario directory added | Does krkn have a matching scenario plugin? | `gh api repos/krkn-chaos/krkn/contents/krkn/scenario_plugins --jq '.[].name'` and look for match |
| `run.sh` references a new Python script or module | Does it exist in krkn's codebase? | `gh api repos/krkn-chaos/krkn/contents/krkn --jq '.[].name'` as needed |
| Base image changed from `:latest` to a pinned tag | Is the pinned version compatible with all env vars used? | Note in review for manual verification |
| Parameter added, renamed, or removed in `krknctl-input.json` or `env.sh` | Does krkn-dashboard's parameterized run form still send the correct env var names? krkn-hub is the source of truth for scenario parameters -- the dashboard must stay in sync | `gh api repos/krkn-chaos/krkn-dashboard/contents/src/components/NewExperiment -H "Accept: application/vnd.github.raw"` and check for the affected parameter name |

### 5c. Triggers for krknctl PRs -> check krkn-hub

| Trigger | What to check | How |
|------|------|------|
| Label regex in `config.json` changed | Do existing krkn-hub Dockerfiles produce labels matching the new regex? | Fetch a sample Dockerfile.template: `gh api repos/krkn-chaos/krkn-hub/contents/pod-scenarios/Dockerfile.template -H "Accept: application/vnd.github.raw"` and extract LABEL lines |
| `ScenarioDataProvider` interface changed | Is the contract with krkn-hub image metadata preserved? | Note in review as high-impact |
| New field added to `typing` package validators | Do krkn-hub's `krknctl-input.json` files use compatible field types? | Fetch a sample: `gh api repos/krkn-chaos/krkn-hub/contents/pod-scenarios/krknctl-input.json -H "Accept: application/vnd.github.raw"` |
| Input field parsing logic changed in `cmd/run.go` | Could this break existing scenario inputs? | Note for manual verification |

### 5d. Triggers for krkn-dashboard PRs -> check krkn, krknctl, and krkn-hub

| Trigger | What to check | How |
|------|------|------|
| Changes to `server/opensearch/` or Elasticsearch query logic | Does the query structure match what krkn actually writes to the index? | `gh api repos/krkn-chaos/krkn/contents/krkn/telemetry -H "Accept: application/vnd.github.raw"` -- check field names and index patterns |
| New experiment type added to the UI (`src/actions/` or `src/components/NewExperiment/`) | Is there a matching krkn scenario plugin? Is it discoverable via krknctl? | `gh api repos/krkn-chaos/krkn/contents/krkn/scenario_plugins --jq '.[].name'` and verify krknctl's label regex would surface it |
| Changes to how the dashboard invokes or parses krknctl output (`server/index.js`) | Does the invocation format match krknctl's CLI contract? | `gh api repos/krkn-chaos/krknctl/contents/cmd -H "Accept: application/vnd.github.raw" --jq '.[].name'` -- check command/flag names haven't drifted |
| Input field schema changes in `src/components/NewExperiment/` | Does the field structure still align with krknctl's `input_fields` label format AND krkn-hub's `krknctl-input.json` definitions? | `gh api repos/krkn-chaos/krknctl/contents/pkg/config/config.json -H "Accept: application/vnd.github.raw"` for field types; fetch a representative `krknctl-input.json` from krkn-hub to verify env var names and defaults match what the dashboard sends |
| Changes to parameterized run logic (how scenario parameters are collected and passed) | Do the parameter names match the env var names in krkn-hub's `env.sh` and `krknctl-input.json`? | `gh api repos/krkn-chaos/krkn-hub/contents/{scenario}/krknctl-input.json -H "Accept: application/vnd.github.raw"` -- krkn-hub is the source of truth for what parameters each scenario accepts |
| Elasticsearch index name or field name changed in `server/` | Could break existing krkn telemetry data already in the index | Note as high-impact; flag for explicit testing |

### 5f. Triggers for other repos -> check krkn-dashboard

| Trigger | What to check | How |
|------|------|------|
| krkn PR changes telemetry output field names or index structure | Does krkn-dashboard's OpenSearch queries still match? | `gh api repos/krkn-chaos/krkn-dashboard/contents/server/opensearch -H "Accept: application/vnd.github.raw"` and inspect field references |
| krknctl PR changes label regex patterns in `config.json` | Does krkn-dashboard's experiment discovery logic still work with the new patterns? | `gh api repos/krkn-chaos/krkn-dashboard/contents/server/index.js -H "Accept: application/vnd.github.raw"` and check how krknctl output is parsed |
| krknctl PR changes CLI flag names or output format for `run` or `list` commands | Does krkn-dashboard's server-side invocation of krknctl still match? | Note for manual verification -- krkn-dashboard calls krknctl as a subprocess |
| krkn-hub PR adds, renames, or removes a parameter in `krknctl-input.json` or `env.sh` | Does krkn-dashboard's parameterized run form still send the correct env var names and defaults? | `gh api repos/krkn-chaos/krkn-dashboard/contents/src/components/NewExperiment -H "Accept: application/vnd.github.raw"` -- check that parameter names haven't drifted |
| krkn-hub PR adds a new scenario directory | Is the new scenario surfaced in krkn-dashboard's experiment selector? | Informational only -- note if the scenario is missing from the UI |

### 5g. When NOT to check cross-repo (skip entirely)

- Documentation-only changes (only `.md` files touched)
- Test-only changes (only `*_test.go` or `tests/` files touched)
- CI/workflow changes (`.github/workflows/` only) unless they change image build/push logic
- Go vendor directory updates (`vendor/` only)
- Comment-only or whitespace-only changes

---

## Step 6: Output the Review Report

Structure your output exactly like this. Only include sections that have findings -- omit empty sections.

```
## PR Review: {repo}#{number} -- {title}

**Author**: {author} | **Size**: +{additions}/-{deletions} across {changedFiles} files | **Base**: {baseRefName}

### Summary
{1-3 sentence overall assessment. Be direct: "Looks good with minor suggestions" or "Has issues that should be addressed before merge" or "Critical: plugin naming violation will cause silent runtime failure". Do not hedge or pad.}

### Critical
{Issues that will cause bugs, runtime failures, or security problems. These should block merge.}
- **[{file}:{line}]** {description}

### Suggestions
{Issues worth fixing but not blocking. Better patterns, missing edge cases, style improvements.}
- **[{file}:{line}]** {description}

### Nits
{Optional improvements. Naming, formatting, minor style. Only include if genuinely helpful.}
- **[{file}:{line}]** {description}

### Cross-Repo Impact
{Only include if cross-repo checks were triggered. State what was checked and findings.}
- **Checked**: {what was verified against which repo}
- **Finding**: {result -- either "aligned" or specific mismatch}

### Checklist
- [{x or space}] Conventional commit title
- [{x or space}] PR description filled in
- [{x or space}] Tests added/updated for new code
- [{x or space}] No security concerns (secrets, injection, unsafe patterns)
- [{x or space}] Documentation PR linked (if user-facing change)
- [{x or space}] CONTRIBUTING.md compliance (single change, squashable)
- [{x or space}] Cross-repo contracts preserved
```

---

## Important Guidelines

### What this skill does NOT do
- **Never post comments on GitHub** -- all output goes to the terminal only. Do not use `gh pr comment`, `gh pr review`, or any command that writes to the PR.
- **Never approve or request changes** on the PR.
- **Never modify code** -- this is a read-only review.
- **Never run tests or linters** -- analyze the diff for likely issues, but do not execute code.
- **Never clone repos for cross-repo checks** -- use `gh api` for lightweight file lookups.
- **Never review vendor/ directory contents** in krknctl -- just note the dependency bump.

### Review philosophy
- **Be specific**: Every finding must reference a file and line (or line range) from the diff.
- **Explain why**: Don't just say "this is wrong" -- explain what will break and how.
- **Prioritize**: Critical > Suggestions > Nits. A review with 1 critical finding and 0 nits is better than one with 0 criticals and 15 nits.
- **Be domain-aware**: You know chaos engineering. A missing rollback, an unbounded blast radius, or a silent plugin loading failure is more important than a style nit.
- **Don't flag what linters catch**: If golangci-lint, flake8, or shellcheck would catch it and the project has CI for it, don't duplicate the effort. Focus on semantic issues that automated tools miss.
- **Respect project conventions**: The project has specific patterns (unittest not pytest, `${VAR:=default}` not `VAR:-default`, `[[ ]]` not `[ ]`). Flag deviations from established patterns, not deviations from generic best practices.

### Cross-repo judgment calls
- If a cross-repo check reveals a mismatch, include it in the review but note whether it is a pre-existing issue or introduced by this PR. Only flag as Critical if the PR introduces the mismatch.
- When fetching cross-repo files via `gh api`, handle 404s gracefully -- the file may not exist yet (e.g., new scenario with no krkn-hub counterpart yet). Note this as informational, not as an error.
- Limit cross-repo API calls to 5 per review to keep the skill fast. Prioritize the highest-signal checks.
