# Testing `everestctl auth login` on a Local Kind Cluster

This guide walks through spinning up a local Kubernetes cluster, deploying Everest,
building the server from source, and testing the `auth login` command end-to-end.

---

## Prerequisites

Make sure the following tools are installed before starting:

| Tool | Why you need it |
|------|----------------|
| `kind` | Creates a Kubernetes cluster inside Docker containers on your machine |
| `kubectl` | The standard CLI for talking to any Kubernetes cluster |
| `helm` | Kubernetes package manager — used to install Everest |
| `docker` | Kind uses Docker under the hood to run cluster nodes |
| `go` | Needed to build the server binary from source |

Check what you have:

```bash
kind version
kubectl version --client
helm version
docker version
go version
```

If `helm` is missing and you don't have sudo access, install it to your local bin:

```bash
curl -sL https://get.helm.sh/helm-v3.21.1-linux-amd64.tar.gz | tar -xz -C /tmp
mv /tmp/linux-amd64/helm ~/.local/bin/helm
```

---

## Step 1 — Create a Kind Cluster

```bash
kind create cluster --name everest
```

**What this does:** Creates a single-node Kubernetes cluster running inside a Docker
container on your machine. Kind stands for "Kubernetes IN Docker." The `--name everest`
flag names the cluster so you can run multiple clusters simultaneously without them
colliding.

After this runs, `kind` automatically updates your `~/.kube/config` and sets the current
context to `kind-everest`, so every `kubectl` command from now on talks to this cluster.

Verify it worked:

```bash
kubectl cluster-info --context kind-everest
```

---

## Step 2 — Install Everest via Helm

Add the Everest Helm chart repository and install the core chart:

```bash
helm repo add openeverest https://openeverest.github.io/helm-charts/
helm repo update
helm install everest-core openeverest/openeverest \
  --devel \
  --version "2.0.0-dev.1" \
  --namespace everest-system \
  --create-namespace
```

**What each flag does:**

- `--devel` — includes pre-release chart versions (anything with a `-dev` or `-rc` suffix)
- `--version "2.0.0-dev.1"` — pins to this exact developer preview release
- `--namespace everest-system` — installs into a dedicated namespace
- `--create-namespace` — creates the namespace if it doesn't already exist

Wait for the pods to come up:

```bash
kubectl get pods -n everest-system
```

You should see something like:

```
NAME                                   READY   STATUS    RESTARTS   AGE
everest-controller-88855bb7f-nnmzq     1/1     Running   0          60s
everest-server-fc7669788-cqbdq         1/1     Running   0          60s
```

Both pods need to show `1/1 Running` before proceeding.

---

## Step 3 — Get the Admin Password

```bash
kubectl get secret everest-accounts -n everest-system \
  -o jsonpath='{.data.users\.yaml}' | base64 --decode
```

**What this does:** Everest stores user accounts in a Kubernetes Secret called
`everest-accounts`. The secret value is base64-encoded YAML. This command decodes it
and prints the raw content, which includes the admin password hash.

Expected output:

```yaml
admin:
  passwordHash: cMRC1T5uaWvwDwkJhiv7FaCfeAcS1X9Zuq4ahqPnbARBglSQOzZdHube2KugPfK5
  enabled: true
  capabilities:
    - login
```

Note the value next to `passwordHash` — that is your admin password. In this developer
preview, passwords are stored in plaintext despite the field being named `passwordHash`.

---

## Step 4 — Why We Need to Build a Local Server Image

The `v2.0.0-dev.1` helm chart ships a pre-built Docker image. That image was built
from the codebase at the point the release was tagged. However, PR #2361 (which added
the `/v1/auth/token` endpoint that `auth login` depends on) was merged into
`release-2.0` *after* that tag was cut.

You can confirm this with:

```bash
git log --oneline --decorate | grep -E "tag:|2361"
```

Output:

```
a6aa8aea Implement API auth token management with refresh tokens (#2361)
32c72249 (tag: v2.0.0-dev.1) update version tag
```

PR #2361 (`a6aa8aea`) is above the tag (`32c72249`), meaning it came after. So the
deployed server simply doesn't know about `POST /v1/auth/token` yet. We need to build
the server from the current source and swap it into the cluster.

---

## Step 5 — Build the Server Binary

The server's Dockerfile is not a multi-stage build that compiles Go — it expects
pre-built binaries in `./bin/`. We build them manually first.

```bash
# The server embeds the frontend at compile time.
# This stub prevents the build from failing if you haven't built the UI.
mkdir -p ./public/dist && [ -f ./public/dist/index.html ] || touch ./public/dist/index.html

# Build the API server binary for Linux (the container OS)
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o ./bin/everest ./cmd

# Build the controller binary (also needed by the Dockerfile)
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o ./bin/manager ./cmd/controller
```

**Why `GOOS=linux GOARCH=amd64`:** Your development machine might be macOS or ARM.
The binary running inside the Docker container must be Linux/amd64 regardless. These
flags cross-compile for the right target.

**Why `CGO_ENABLED=0`:** The base image in the Dockerfile is `scratch` (completely
empty). CGO produces binaries that depend on the system's C library (`glibc`), but
`scratch` has no libraries at all. Disabling CGO produces a fully static binary that
runs without any dependencies.

---

## Step 6 — Build the Docker Image

```bash
docker build \
  -t everest-server:local \
  -f build/package/server/Dockerfile \
  --target openeverest \
  .
```

**What each part does:**

- `-t everest-server:local` — names and tags the image so we can refer to it later
- `-f build/package/server/Dockerfile` — the Dockerfile lives inside `build/package/server/`, not in the repo root
- `--target openeverest` — the Dockerfile has multiple build targets (`openeverest` for the API server, `controller` for the controller). We only want the server.
- `.` — the build context is the repo root, so Docker can find `./bin/everest`

---

## Step 7 — Load the Image into Kind

```bash
kind load docker-image everest-server:local --name everest
```

**Why this step exists:** Kind clusters are isolated — they run inside their own Docker
network and cannot pull images from your local Docker daemon the way a regular `docker run`
would. `kind load docker-image` explicitly copies the image from your local Docker into
the kind cluster's internal image store so that pods can use it.

---

## Step 8 — Patch the Deployment

Replace the running server image with the locally built one:

```bash
kubectl set image deployment/everest-server everest=everest-server:local -n everest-system
```

The container inside the deployment is named `everest` (not `everest-server`). If you
are ever unsure of container names, check with:

```bash
kubectl get deployment everest-server -n everest-system \
  -o jsonpath='{.spec.template.spec.containers[*].name}'
```

Wait for the rollout to complete:

```bash
kubectl rollout status deployment/everest-server -n everest-system
```

Expected output:

```
deployment "everest-server" successfully rolled out
```

---

## Step 9 — Start Port-Forwarding

The Everest server pod is inside the cluster and not directly reachable from your
machine. Port-forwarding creates a tunnel from `localhost:8080` to the service inside
the cluster:

```bash
kubectl port-forward svc/everest 8080:8080 -n everest-system &
```

The `&` runs it in the background so your terminal stays usable.

**Important:** When the server pod is replaced (Step 8), any existing port-forward
process dies automatically because it was connected to the old pod. Always restart
port-forwarding after patching the deployment.

Verify the tunnel is working:

```bash
curl -s http://localhost:8080/v1/version -H "Authorization: Bearer test"
```

Expected response (the JWT is invalid, but the server is reachable and responding):

```json
{"message":"invalid or expired jwt"}
```

---

## Step 10 — Build the CLI

```bash
go build -o /tmp/everestctl ./cmd/cli
```

This builds the `everestctl` binary locally. We output it to `/tmp` to keep it separate
from the repo. The server binary at `./cmd` is the API server; `./cmd/cli` is the CLI.

---

## Step 11 — Test `auth login`

```bash
/tmp/everestctl auth login \
  --server http://localhost:8080 \
  --username admin \
  --password <your-password-from-step-3>
```

**Expected output:**

```
✅ Logged in to http://localhost:8080 as admin
```

---

## Step 12 — Verify the Config File

```bash
cat ~/.config/everest/config.yaml
```

Expected output (tokens will differ):

```yaml
apiVersion: v1
kind: Config
currentContext: admin@localhost:8080
contexts:
    - name: admin@localhost:8080
      context:
        server: http://localhost:8080
        accessToken: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
        refreshToken: everest_rt_c1b42351509c4971bf47499888c5dd5b_EJwlm...
        expiresAt: 2026-06-15T11:07:54.338343412+05:30
```

**What each field means:**

- `currentContext` — which context is active by default (`username@host:port` format)
- `accessToken` — a short-lived JWT (15 minutes). Used in API requests as a Bearer token.
- `refreshToken` — a long-lived opaque token (30 days), prefixed `everest_rt_`. Used to
  get a new access token when the current one expires.
- `expiresAt` — when the access token expires. Future commands can check this before
  making API calls and call `Refresh()` if needed.

Check file permissions:

```bash
ls -la ~/.config/everest/config.yaml
```

Expected:

```
-rw------- 1 anya anya 1247 Jun 15 10:52 /home/anya/.config/everest/config.yaml
```

The file must be `0600` (`-rw-------`). This means only your user can read or write it.
The tokens stored here grant full API access, so they should never be world-readable.

---

## Summary

| Step | What happened |
|------|--------------|
| 1 | Created a local Kubernetes cluster with kind |
| 2 | Installed Everest `v2.0.0-dev.1` via Helm |
| 3 | Retrieved the admin password from a Kubernetes Secret |
| 4 | Identified that the released image predates the auth endpoint |
| 5 | Built server and controller binaries from source |
| 6 | Packaged the binaries into a Docker image |
| 7 | Loaded the image into the kind cluster |
| 8 | Patched the deployment to use the new image |
| 9 | Port-forwarded the server to localhost:8080 |
| 10 | Built the CLI binary |
| 11 | Ran `everestctl auth login` and got a success response |
| 12 | Verified tokens were written to `~/.config/everest/config.yaml` with correct permissions |
