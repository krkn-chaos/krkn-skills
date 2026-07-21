---
name: krkn-operator-local
description: >
  Check out a krkn-operator-console PR (and optionally a krkn-operator PR),
  build the operator backend locally, start the frontend dev server, and
  return the local URL to test the changes. Use this skill when the user wants
  to spin up a local test environment for a krkn-operator-console pull request,
  run an operator PR locally, or verify UI changes against a live backend.
user_invocable: true
arguments:
  - name: console_pr
    description: >
      krkn-operator-console PR in any of these formats:
      - Full URL: https://github.com/krkn-chaos/krkn-operator-console/pull/42
      - Short ref: krkn-chaos/krkn-operator-console#42
      - Number only: #42 or 42 (assumes krkn-chaos/krkn-operator-console)
    required: true
  - name: operator_pr
    description: >
      (Optional) krkn-operator PR to check out alongside the console PR.
      Same formats as console_pr but for the krkn-chaos/krkn-operator repo.
      If omitted, the main branch of the cloned krkn-operator repo is used.
    required: false
---

# krkn-operator Local Test Environment

You are a developer-experience assistant for the krkn-chaos ecosystem. Your job is to:

1. Check out the PR branches for `krkn-operator-console` (and optionally `krkn-operator`)
2. Build the Go operator backend
3. Create a one-node local Kubernetes cluster and install the operator CRDs
4. Start the console frontend dev server
5. Report the local URL so the user can test immediately

The two components talk to each other:

```
Browser → http://localhost:3000  (Vite dev server, krkn-operator-console)
              ↓ /api/* proxied to
           http://localhost:8080  (Go REST API, krkn-operator)
```

---

## Step 0: Locate the krkn-skills Root and Set Up Repos

Find the root of the `krkn-skills` repo — repos are always cloned there so the user's existing local checkouts are never touched:

```bash
# Find krkn-skills root by walking up from the skill file, or search common locations
KRKN_SKILLS_DIR=$(
  git -C "$(dirname "$0")" rev-parse --show-toplevel 2>/dev/null ||
  for BASE in "$HOME/Github/krkn-chaos" "$HOME/Github" "$HOME/Projects" "$HOME/src"; do
    [ -d "$BASE/krkn-skills" ] && echo "$BASE/krkn-skills" && break
  done
)

if [ -z "$KRKN_SKILLS_DIR" ]; then
  echo "ERROR: Could not locate krkn-skills repo. Clone it first:"
  echo "  git clone https://github.com/krkn-chaos/krkn-skills ~/Github/krkn-chaos/krkn-skills"
  exit 1
fi

REPOS_DIR="$KRKN_SKILLS_DIR/repos"
CONSOLE_DIR="$REPOS_DIR/krkn-operator-console"
OPERATOR_DIR="$REPOS_DIR/krkn-operator"

mkdir -p "$REPOS_DIR"
```

Clone the repos into `krkn-skills/repos/` if they are not already there:

```bash
if [ ! -d "$CONSOLE_DIR/.git" ]; then
  echo "Cloning krkn-operator-console into $CONSOLE_DIR ..."
  gh repo clone krkn-chaos/krkn-operator-console "$CONSOLE_DIR"
fi

if [ ! -d "$OPERATOR_DIR/.git" ]; then
  echo "Cloning krkn-operator into $OPERATOR_DIR ..."
  gh repo clone krkn-chaos/krkn-operator "$OPERATOR_DIR"
fi
```

If `repos/` is not in `.gitignore` for krkn-skills, add it to avoid accidentally committing checked-out code:

```bash
grep -qxF 'repos/' "$KRKN_SKILLS_DIR/.gitignore" 2>/dev/null || echo 'repos/' >> "$KRKN_SKILLS_DIR/.gitignore"
```

Use `$CONSOLE_DIR` and `$OPERATOR_DIR` for all subsequent steps.

---

## Step 1: Parse PR References

Parse `{{ console_pr }}` to extract owner, repo, and PR number:

- `https://github.com/krkn-chaos/krkn-operator-console/pull/42` → number=42
- `krkn-chaos/krkn-operator-console#42` → number=42
- `#42` or `42` → number=42, repo=krkn-operator-console

If `{{ operator_pr }}` was provided, parse it the same way for `krkn-operator`.

---

## Step 2: Fetch PR Branch Names

For each PR, get the head branch name:

```bash
# Console PR
CONSOLE_BRANCH=$(gh pr view {{ console_pr_number }} \
  --repo krkn-chaos/krkn-operator-console \
  --json headRefName --jq '.headRefName')
CONSOLE_AUTHOR=$(gh pr view {{ console_pr_number }} \
  --repo krkn-chaos/krkn-operator-console \
  --json author --jq '.author.login')

# Operator PR (if provided)
OPERATOR_BRANCH=$(gh pr view {{ operator_pr_number }} \
  --repo krkn-chaos/krkn-operator \
  --json headRefName --jq '.headRefName')
OPERATOR_AUTHOR=$(gh pr view {{ operator_pr_number }} \
  --repo krkn-chaos/krkn-operator \
  --json author --jq '.author.login')
```

If the operator PR is from a fork (author != krkn-chaos member), add the fork remote before checking out:

```bash
FORK_URL=$(gh pr view {{ operator_pr_number }} \
  --repo krkn-chaos/krkn-operator \
  --json headRepositoryOwner,headRepository \
  --jq '"https://github.com/\(.headRepositoryOwner.login)/\(.headRepository.name)"')
git -C "$OPERATOR_DIR" remote add "$OPERATOR_AUTHOR" "$FORK_URL" 2>/dev/null || true
git -C "$OPERATOR_DIR" fetch "$OPERATOR_AUTHOR" "$OPERATOR_BRANCH"
```

Do the same fork handling for the console PR if needed.

---

## Step 3: Check Out the Branches

**Stash any uncommitted changes first** so the user does not lose work:

```bash
# Console repo
cd "$CONSOLE_DIR"
git status --short | grep -q . && git stash push -u -m "krkn-operator-local: auto-stash before PR checkout"
git fetch origin
git checkout "$CONSOLE_BRANCH" 2>/dev/null || git checkout -b "$CONSOLE_BRANCH" "origin/$CONSOLE_BRANCH"
git pull --ff-only 2>/dev/null || true  # no-op if already up to date
```

If an operator PR was provided:

```bash
# Operator repo
cd "$OPERATOR_DIR"
git status --short | grep -q . && git stash push -u -m "krkn-operator-local: auto-stash before PR checkout"
git fetch origin
git checkout "$OPERATOR_BRANCH" 2>/dev/null || git checkout -b "$OPERATOR_BRANCH" "origin/$OPERATOR_BRANCH"
git pull --ff-only 2>/dev/null || true
```

Report what branch each repo is now on.

---

## Step 4: Dependency Check

Check that required tools are available:

```bash
command -v go       || { echo "ERROR: go not found. Install Go from https://go.dev/dl/"; exit 1; }
command -v node     || { echo "ERROR: node not found. Install Node.js from https://nodejs.org/"; exit 1; }
command -v npm      || { echo "ERROR: npm not found. Comes with Node.js."; exit 1; }
command -v kubectl  || { echo "ERROR: kubectl not found. Install from https://kubernetes.io/docs/tasks/tools/"; exit 1; }
command -v kind     || command -v minikube || echo "WARN: neither kind nor minikube found — cluster creation will be skipped. Install kind: brew install kind"
```

Check Go version against go.mod minimum:

```bash
GO_REQUIRED=$(head -5 "$OPERATOR_DIR/go.mod" | grep '^go ' | awk '{print $2}')
GO_ACTUAL=$(go version | awk '{print $3}' | sed 's/go//')
echo "Go required: $GO_REQUIRED | Go installed: $GO_ACTUAL"
```

If Go version is insufficient, warn and ask the user whether to continue.

---

## Step 5: Build the Operator Backend

```bash
cd "$OPERATOR_DIR"
echo "=== Building krkn-operator ==="
make build 2>&1
```

`make build` compiles `./cmd/main.go` into `./bin/manager`. If it fails:

- Show the last 30 lines of build output
- Tell the user what failed
- Stop here (do not start servers if the build is broken)

If the build succeeds, report: `krkn-operator built successfully → $OPERATOR_DIR/bin/manager`

---

## Step 6: Create a One-Node Local Cluster

Create a single-node Kubernetes cluster named `krkn-operator-local` for the operator to connect to. Prefer `kind`; fall back to `minikube`.

### 6a. Check if the cluster already exists

```bash
CLUSTER_NAME="krkn-operator-local"

if command -v kind &>/dev/null; then
  CLUSTER_TOOL="kind"
  CLUSTER_EXISTS=$(kind get clusters 2>/dev/null | grep -q "^${CLUSTER_NAME}$" && echo "yes" || echo "no")
elif command -v minikube &>/dev/null; then
  CLUSTER_TOOL="minikube"
  CLUSTER_EXISTS=$(minikube status -p "$CLUSTER_NAME" &>/dev/null && echo "yes" || echo "no")
else
  echo "WARN: Neither kind nor minikube found — skipping cluster creation."
  echo "      Install kind with: brew install kind"
  CLUSTER_TOOL="none"
  CLUSTER_EXISTS="no"
fi
```

If the cluster already exists, reuse it (skip creation). Print which tool and cluster will be used.

### 6b. Create the cluster (if not already present)

**kind:**

Write a kind config that maps the operator's host port (8080) into the cluster node, so pods inside the cluster can call back to the operator. Also detect whether Docker or Podman is the backing runtime — the host alias differs between them.

```bash
if [ "$CLUSTER_TOOL" = "kind" ] && [ "$CLUSTER_EXISTS" = "no" ]; then
  echo "=== Creating one-node kind cluster: $CLUSTER_NAME ==="

  # Detect container runtime to set the correct host alias
  if podman info &>/dev/null 2>&1; then
    CONTAINER_RUNTIME="podman"
    HOST_ALIAS="host.containers.internal"
  else
    CONTAINER_RUNTIME="docker"
    HOST_ALIAS="host.docker.internal"
  fi
  echo "Container runtime: $CONTAINER_RUNTIME (host alias: $HOST_ALIAS)"

  # Resolve the host IP for the alias
  HOST_IP=$(ip route 2>/dev/null | awk '/default/ {print $3; exit}' || \
            route -n get default 2>/dev/null | awk '/gateway/ {print $2; exit}' || \
            echo "172.17.0.1")
  echo "Host IP: $HOST_IP"

  # Write kind cluster config with extraPortMappings and host alias patch
  KIND_CONFIG=$(mktemp /tmp/kind-config-XXXXXX.yaml)
  cat > "$KIND_CONFIG" <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 8080
    hostPort: 8080
    protocol: TCP
  - containerPort: 3000
    hostPort: 3000
    protocol: TCP
EOF

  kind create cluster --name "$CLUSTER_NAME" --config "$KIND_CONFIG" --wait 60s
  rm -f "$KIND_CONFIG"
  echo "Cluster created."

  # Patch CoreDNS or /etc/hosts in each node to resolve the host alias
  for NODE in $(kind get nodes --name "$CLUSTER_NAME"); do
    # Add host alias to /etc/hosts inside the kind node container
    if [ "$CONTAINER_RUNTIME" = "podman" ]; then
      podman exec "$NODE" sh -c "echo '$HOST_IP $HOST_ALIAS' >> /etc/hosts" 2>/dev/null || true
    else
      docker exec "$NODE" sh -c "echo '$HOST_IP $HOST_ALIAS' >> /etc/hosts" 2>/dev/null || true
    fi
    echo "Patched /etc/hosts in node $NODE: $HOST_IP → $HOST_ALIAS"
  done
fi
```

**minikube fallback:**

```bash
if [ "$CLUSTER_TOOL" = "minikube" ] && [ "$CLUSTER_EXISTS" = "no" ]; then
  echo "=== Creating one-node minikube cluster: $CLUSTER_NAME ==="
  minikube start -p "$CLUSTER_NAME" --nodes=1 --driver=docker \
    --ports=8080:8080,3000:3000
  echo "Cluster created."
fi
```

If cluster creation fails, warn the user and continue — the operator REST API can still serve some endpoints without a live cluster.

### 6c. Set kubectl context

```bash
if [ "$CLUSTER_TOOL" = "kind" ]; then
  kubectl config use-context "kind-${CLUSTER_NAME}"
elif [ "$CLUSTER_TOOL" = "minikube" ]; then
  kubectl config use-context "$CLUSTER_NAME"
fi

echo "kubectl context: $(kubectl config current-context)"
kubectl cluster-info --context "$(kubectl config current-context)" 2>&1 | head -3
```

### 6d. Create namespace, install CRDs, and apply RBAC

```bash
if [ "$CLUSTER_TOOL" != "none" ]; then
  echo "=== Creating krkn-operator-system namespace ==="
  kubectl create namespace krkn-operator-system 2>/dev/null || echo "Namespace already exists."

  # Generate CRDs and RBAC manifests if not already present
  ls "$OPERATOR_DIR/config/crd/bases/"*.yaml &>/dev/null || {
    echo "CRD files not found — running make manifests to generate them..."
    cd "$OPERATOR_DIR" && make manifests 2>&1
  }

  echo "=== Installing CRDs and RBAC via kustomize (excluding Deployment) ==="
  # Render the full default overlay and apply everything — the Deployment will fail
  # to pull images but that is expected since we run the operator locally with make run.
  kubectl kustomize "$OPERATOR_DIR/config/default" | \
    kubectl apply -f - --validate=false 2>&1 | grep -v "^error.*image\|^Error.*image" || true

  echo "CRDs installed: $(kubectl get crds 2>/dev/null | grep -c krkn || echo 0)"
  echo "Service accounts:"
  kubectl get sa -n krkn-operator-system 2>/dev/null
fi
```

Report the cluster name, context, CRD count, and service accounts created.

---

## Step 7: Install Console Dependencies (if needed)

```bash
cd "$CONSOLE_DIR"
if [ ! -d node_modules ] || [ package.json -nt node_modules ]; then
  echo "=== Installing npm dependencies ==="
  npm install 2>&1
fi
```

---

## Step 8: Start the Servers

Launch the operator and the console in separate terminal windows automatically. Use tmux if available (preferred — splits a single window into two visible panes); otherwise open two macOS Terminal windows via osascript.

### 8a. Check for port conflicts first

```bash
OPERATOR_PORT_PID=$(lsof -ti:8080 2>/dev/null || true)
CONSOLE_PORT_PID=$(lsof -ti:3000 2>/dev/null || true)

if [ -n "$OPERATOR_PORT_PID" ]; then
  echo "WARNING: Port 8080 is already in use by PID $OPERATOR_PORT_PID"
  echo "Kill it? (y/n)"
  read -r REPLY
  [ "$REPLY" = "y" ] && kill -9 "$OPERATOR_PORT_PID" && echo "Port 8080 cleared"
fi

if [ -n "$CONSOLE_PORT_PID" ]; then
  echo "WARNING: Port 3000 is already in use by PID $CONSOLE_PORT_PID"
  echo "Kill it? (y/n)"
  read -r REPLY
  [ "$REPLY" = "y" ] && kill -9 "$CONSOLE_PORT_PID" && echo "Port 3000 cleared"
fi
```

### 8b. Launch in tmux (preferred) or Terminal windows

```bash
if command -v tmux &>/dev/null; then
  # Kill any existing session from a previous run
  tmux kill-session -t krkn-operator-local 2>/dev/null || true

  # Pane 0: operator backend
  tmux new-session -d -s krkn-operator-local -x 220 -y 50
  tmux rename-window -t krkn-operator-local 'krkn-local'
  tmux send-keys -t krkn-operator-local "cd '$OPERATOR_DIR' && make run" Enter

  # Pane 1 (right split): console frontend
  tmux split-window -h -t krkn-operator-local
  tmux send-keys -t krkn-operator-local "cd '$CONSOLE_DIR' && npm run dev" Enter

  echo "Servers launching in tmux session 'krkn-operator-local'."
  echo "Attaching now (Ctrl-b d to detach without stopping servers)..."
  sleep 1
  tmux attach -t krkn-operator-local

elif [[ "$OSTYPE" == "darwin"* ]]; then
  # macOS fallback: open two Terminal windows
  osascript -e "tell app \"Terminal\" to do script \"cd '$OPERATOR_DIR' && make run\""
  sleep 1
  osascript -e "tell app \"Terminal\" to do script \"cd '$CONSOLE_DIR' && npm run dev\""
  echo "Opened two Terminal windows. Watch them for startup messages."

else
  # Linux fallback without tmux
  echo "tmux not found. Install it for automatic terminal splitting:"
  echo "  brew install tmux   (macOS)"
  echo "  sudo apt install tmux   (Ubuntu/Debian)"
  echo ""
  echo "To start manually:"
  echo "  Terminal 1:  cd '$OPERATOR_DIR' && make run"
  echo "  Terminal 2:  cd '$CONSOLE_DIR' && npm run dev"
  exit 0
fi
```

### 8c. Wait for both servers to be ready

After launching, poll until both ports respond (up to 60 seconds each):

```bash
echo "Waiting for operator API on :8080 ..."
for i in $(seq 1 60); do
  curl -sf http://localhost:8080/api/v1/auth/login &>/dev/null && break
  sleep 1
done
curl -sf http://localhost:8080/api/v1/auth/login &>/dev/null || echo "WARN: operator may not be fully ready yet"

echo "Waiting for console dev server on :3000 ..."
for i in $(seq 1 60); do
  curl -sf http://localhost:3000 &>/dev/null && break
  sleep 1
done
curl -sf http://localhost:3000 &>/dev/null || echo "WARN: console may not be fully ready yet"
```

### 8d. Register and log in the admin user

Once the operator is up, always run these two commands automatically:

```bash
echo "=== Registering admin user ==="
curl -sf -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"userId":"admin@local.dev","password":"Admin1234!","name":"Admin","surname":"User","role":"admin"}' \
  && echo "Admin user registered." \
  || echo "WARN: register failed (user may already exist — continuing)"

echo "=== Logging in ==="
LOGIN_RESPONSE=$(curl -sf -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"userId":"admin@local.dev","password":"Admin1234!"}')

if [ -n "$LOGIN_RESPONSE" ]; then
  echo "Login successful."
  echo "$LOGIN_RESPONSE" | grep -o '"token":"[^"]*"' || true
else
  echo "WARN: login failed — check the operator terminal for errors"
fi
```

If register returns a 409 (user already exists), that is not an error — just proceed.

### 8e. Print the ready summary

```
═══════════════════════════════════════════════════════════════════════
 krkn-operator-console local environment ready
═══════════════════════════════════════════════════════════════════════

 Console PR : krkn-chaos/krkn-operator-console#{{ console_pr_number }}
              branch: {{ console_branch }}
 Operator   : {{ if operator_pr }}krkn-chaos/krkn-operator#{{ operator_pr_number }} (branch: {{ operator_branch }}){{ else }}main{{ end }}
 Cluster    : krkn-operator-local ({{ cluster_tool }}, context: {{ kubectl_context }})
              host alias: {{ host_alias }} → {{ host_ip }}

 TEST URL   → http://localhost:3000
 Auth       : admin user registered and logged in automatically ✓

 API calls from the console proxy to http://localhost:8080 via vite.config.ts

───────────────────────────────────────────────────────────────────────
 STOP SERVERS
───────────────────────────────────────────────────────────────────────

  tmux kill-session -t krkn-operator-local    # if using tmux
  # or just close the Terminal windows

───────────────────────────────────────────────────────────────────────
 DELETE CLUSTER  (when done testing)
───────────────────────────────────────────────────────────────────────

  kind delete cluster --name krkn-operator-local       # if using kind
  minikube delete -p krkn-operator-local               # if using minikube

───────────────────────────────────────────────────────────────────────
 START FRESH  (optional — wipe clones and re-clone next run)
───────────────────────────────────────────────────────────────────────

  rm -rf '{{ REPOS_DIR }}'
═══════════════════════════════════════════════════════════════════════
```

---

## Important Guidelines

- **Use tmux or osascript to launch** — do not run `make run` or `npm run dev` directly in a tool call (they block forever).
- **Always show which branch each repo is on** after checkout, so the user can verify.
- **If the build fails**, stop before launching any servers — show the last 30 lines of build output.
- **If kubectl is missing**, stop — it is required for cluster creation and CRD installation.
- **If kind and minikube are both missing**, warn the user and skip cluster creation — the operator REST API can still run, but chaos execution will fail without a cluster.
- **Reuse existing clusters** — if `krkn-operator-local` already exists, skip creation and just set the context. Never delete an existing cluster without asking.
- **Port forwarding / host alias** — always patch `/etc/hosts` inside each kind node so pods can reach the host operator via `host.containers.internal` (podman) or `host.docker.internal` (docker). Use `extraPortMappings` in the kind config so ports 8080 and 3000 are mapped from the node into the host.
- **Do not modify any source files** — this skill checks out code and builds it; it does not review or edit it.
