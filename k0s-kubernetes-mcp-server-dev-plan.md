# Plan: Deploy forked `kubernetes-mcp-server` into local k0s (dev)

This document captures the current plan, assumptions, and decisions made during this session so it can be reused and extended. Scope is “simplest working deployment” only: no Istio Ambient, no AgentGateway, no ingress.

## Goals
- Run a **forked** `kubernetes-mcp-server` build in the local k0s cluster.
- Deploy into its own namespace: `mcp-system`.
- Use **in-cluster auth** (ServiceAccount + RBAC), not kubeconfig in the pod.
- Provide **broad (for now) read access** across *all namespaces* so the AI triage agent can diagnose.
- Make the MCP server reachable from the laptop over WireGuard by exposing a **non-standard port** on the node.
- Use the upstream Helm chart if possible (avoid modifying it).
- Provide **repeatable, idempotent** setup + teardown scripts.
- Publish images to a **globally accessible registry** for future scale testing.

## Inputs & decisions (session log)
- Cluster: single-node k0s (controller+worker) on a host reachable from the laptop via WireGuard VPN.
- Access needs:
  - MCP server must be able to see **all namespaces** for now.
  - k0s node(s) have outbound internet access (registry pulls are feasible).
  - Helm release Secrets/ConfigMaps access is OK to grant broadly for now; we’ll tighten later.
- Server invocation currently used locally: `./kubernetes-mcp-server --port 8686 --log-level 2`.
  - Assumption: upstream binary already supports `--port` (or your fork does); we will wire Helm values to pass it.
- “Random port” requirement:
  - Port must be reachable from laptop **without relying on `kubectl port-forward`** (no routing/forwarding path in your current WireGuard setup).
  - Decision: expose via **Service type `NodePort`**.
  - Helm chart validation:
    - The upstream chart uses `.Values.service.port` for both the **containerPort** and the app config `config.port`.
    - The chart does **not** expose a `service.nodePort` value in `values.yaml`; the Service template only sets `type` and `port`.
    - Therefore, we will rely on Kubernetes to **auto-allocate** a NodePort (random within the cluster’s NodePort range) and have the setup script **read and print** the assigned nodePort.
  - The key laptop-reachable port is the **Service NodePort** on the node’s WireGuard-reachable IP.
- Script location clarification:
  - Scripts will run on the k0s node (or any machine with admin `kubectl` + `helm` configured for that cluster), not necessarily the laptop.
- Image distribution:
  - Decision: build and push to a globally accessible registry (e.g., GHCR), then configure Helm to pull it.

## High-level architecture
- `kubernetes-mcp-server` runs as a Deployment in `mcp-system`.
- Pod authenticates to Kubernetes API using in-cluster config:
  - `serviceAccountName: kubernetes-mcp-server`
  - token/CA injected automatically at `/var/run/secrets/kubernetes.io/serviceaccount/*`
- Network access from laptop:
  - `NodeIP (WireGuard reachable) : NodePort (random)` → Service → Pod (container port)
- No LoadBalancer, no Ingress, no mesh.

## RBAC strategy (broad now, tighten later)
### Baseline
- Create `ServiceAccount` in `mcp-system`: `kubernetes-mcp-server`.
- Bind built-in `ClusterRole` `view` to that ServiceAccount via `ClusterRoleBinding` (cluster-wide read of common resources).
  - Note: `view` does **not** include `nodes` (cluster-scoped). If you need “faults” / cluster health signals that watch Nodes, add a separate ClusterRole for `nodes` `get/list/watch`.

### Helm read-only (broad now)
- Add a `ClusterRole` (e.g., `helm-readonly`) that allows reading Helm’s release storage:
  - `secrets` and `configmaps`: `get`, `list`, `watch`
- Bind it cluster-wide via `ClusterRoleBinding`.

### Later tightening (explicitly out-of-scope for this phase)
- Replace cluster-wide `view` with a narrower ClusterRole (only the resource kinds MCP server actually needs).
- Constrain Helm release access to selected namespaces and/or labels (if feasible for your Helm usage model).
- Consider a dedicated Helm MCP server with a gated surface (future design).

## Helm chart approach (don’t modify unless forced)
We will attempt to use the upstream chart and override only values:
- Image: repository/tag
- ServiceAccount: create/use existing
- Service exposure: `NodePort` (auto-assigned nodePort)
- Port/logging: set `.Values.service.port` (drives container port + config port) and use config (or additional config keys) for log level
- Ingress disabled

If the chart does not expose required knobs (ports/args/service type), then the fallback is:
1) Use `helm --set ...` / values override if possible, or
2) Vendor the chart locally with minimal changes (only to expose needed values), documenting the delta.

## Repeatable scripts (idempotent)
Create two scripts in a predictable place on the node (suggested: `~/global-bin/`):

### 1) `mcp-k8s-up`
Responsibilities:
- Ensure prerequisites exist:
  - `kubectl` configured to talk to the k0s cluster
  - `helm` available
- Create/ensure namespace `mcp-system`.
- Apply RBAC manifests (ServiceAccount + ClusterRoleBindings + Helm read-only ClusterRole/Binding).
- Use **NodePort auto-allocation**:
  - Deploy Service as `type: NodePort` (no explicit nodePort set).
  - Read the assigned nodePort from the Service and print it for laptop use.
- Deploy via Helm (idempotent):
  - `helm upgrade --install kubernetes-mcp-server <chart> -n mcp-system -f <generated-values>`
- Print connection info:
  - Node IP to use (WireGuard-reachable): either from input env var (recommended) or detected.
  - URL: `http://<NODE_WG_IP>:<NODE_PORT>/...` (path depends on server).

Suggested knobs (env vars or flags):
- `--node-ip <ip>` (recommended; avoids unreliable autodetect)
- `--image ghcr.io/<org>/<repo>:<tag>`
- `--chart <oci-or-path>`
- `--nodeport-range 30000-32767` (default)
- `--release kubernetes-mcp-server` (default)
- `--namespace mcp-system` (default)

Idempotency rules:
- `kubectl apply -f ...` for manifests.
- Use `helm upgrade --install` with the same release name.

### 2) `mcp-k8s-down`
Responsibilities (idempotent and safe by default):
- `helm uninstall kubernetes-mcp-server -n mcp-system` (ignore if not present).
- Delete RBAC objects created by the plan (ignore-not-found).
- Delete the NodePort ConfigMap (optional; default delete so next run re-generates a random port).
- Optionally delete the namespace (`--delete-namespace` flag).

## Container image build & publish (Mac-friendly)
Target: build from your local fork checkout on your Mac and push to a global registry (GHCR recommended).

### Option A: GHCR (recommended)
Prereqs:
- A GitHub account/org with GHCR enabled.
- `docker` installed on macOS.
- `gh` CLI optional but helpful.

Build and push (template):
1) Authenticate:
   - `echo $GHCR_TOKEN | docker login ghcr.io -u <github-username> --password-stdin`
2) Build:
   - `docker build -t ghcr.io/<org-or-user>/kubernetes-mcp-server:<tag> <path-to-fork>`
3) Push:
   - `docker push ghcr.io/<org-or-user>/kubernetes-mcp-server:<tag>`

Tagging convention (recommended):
- `dev-<gitsha>` (unique, traceable)
- Optionally also push `dev-latest` for convenience during iteration

### Option B: Multi-arch (future-proofing)
If your k0s node is `linux/amd64` and your Mac is Apple Silicon, you may need multi-arch:
- `docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/... --push .`

## Validation (manual, using MCP Inspector from laptop)
1) Run `mcp-k8s-up` on the node with explicit `--node-ip <wireguard-ip>`.
2) From laptop, connect MCP Inspector to `http://<node-wg-ip>:<nodeport>` (exact MCP endpoint path per server).
3) Confirm the server can read across namespaces as expected (via inspector-driven queries).
4) Spot-check that writes are forbidden (if server exposes any write-capable tools, ensure they’re disabled or RBAC blocks them).

## Open items to resolve during implementation (next stage)
- Confirm upstream chart values for:
  - container port / service targetPort
  - extra args (to set `--port` and `--log-level`)
  - service type and fixed `nodePort`
  - serviceAccount creation toggle + name
- Confirm the MCP server HTTP path(s) used by MCP Inspector for this implementation (health, mcp endpoint).
- Decide whether to:
  - enforce host firewall rules to limit NodePort to the WireGuard CIDR (recommended hardening), and/or
  - bind to specific interfaces if relevant.

## Incident notes (what we learned)
- Initial deployment used `service.port=8686` which made the chart set containerPort + probes to 8686.
- The container (image `ghcr.io/randybias/kubernetes-mcp-server-with-events:dev-latest`) still started HTTP on **8080** (`http.go:60`), so probes failed with `connection refused` and the pod CrashLooped.
- The chart’s port is driven by config (`config.port`) + containerPort, but this image appears to ignore `config.port` (or uses a different setting for the HTTP listen port).
- Minimal fix: deploy with `service.port=8080` (keeping external access via random NodePort). This produced a Ready pod and a random NodePort for laptop access.

## Transport troubleshooting notes (MCP Inspector)
- Symptom observed from laptop: MCP Inspector fails to maintain/subscribe via streamable HTTP with errors like `Failed to reconnect SSE stream ... code: 409 (Conflict)`.
- Root cause hint: the Go MCP SDK uses SSE streams under the hood for streamable HTTP. A `409 Conflict` can be returned when a **stream ID conflicts with an ongoing stream** (i.e., the server still believes a previous SSE stream is open).
- Common reason in VPN/NAT setups: **idle TCP connections can be dropped without a clean close (no FIN/RST)**. The client reconnects, but the server hasn’t detected the prior stream is dead (no keepalives/traffic), so it rejects the new stream with `409`.
- Mitigations (choose one):
  - Configure WireGuard `PersistentKeepalive = 25` on the laptop peer (recommended).
  - Ensure the client sends periodic MCP keepalive requests (if supported by the client).
  - Add server-side SSE heartbeat/keepalive behavior in the fork (future improvement).
