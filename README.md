# kubernetes-mcp-server-with-events on k0s (deployment)

Deploy `ghcr.io/randybias/kubernetes-mcp-server-with-events` into a k0s Kubernetes cluster using the upstream `kubernetes-mcp-server` Helm chart, with:
- in-cluster auth (ServiceAccount + RBAC)
- its own namespace (`mcp-system`)
- NodePort exposure (so a laptop can reach it over WireGuard/VPN)
- repeatable `up` / `down` scripts

This repo intentionally keeps things simple: no Istio, no ingress, no external auth on the MCP server.

## Prerequisites (on the k0s node or any admin host)
- `kubectl` configured with admin access to the cluster
- `helm` (v3+)
- Network access from cluster nodes to pull images from GHCR

## Files
- `mcp-k8s-up`: idempotent install/upgrade (RBAC + Helm deploy)
- `mcp-k8s-down`: idempotent teardown (Helm uninstall + RBAC cleanup)
- `mcp-system-rbac.yaml`: namespace + ServiceAccount + RBAC (includes Nodes read for `faults` mode)
- `k0s-kubernetes-mcp-server-dev-plan.md`: session notes, decisions, and follow-ups

## Quick start

1) Decide the image tag you want to deploy

Example:
```bash
IMAGE=ghcr.io/randybias/kubernetes-mcp-server-with-events:dev-latest
```

2) Run the deployment script

Pass the WireGuard/VPN-reachable node IP so it prints a URL you can use from your laptop:
```bash
./mcp-k8s-up --image "$IMAGE" --node-ip <WIREGUARD_NODE_IP>
```

The script:
- applies `mcp-system-rbac.yaml`
- installs/upgrades the upstream Helm chart `oci://ghcr.io/containers/charts/kubernetes-mcp-server`
- exposes the service as `type: NodePort` (Kubernetes auto-assigns a random nodePort)
- prints the assigned NodePort and a laptop URL

3) Connect from your laptop

Use the printed URL and connect MCP Inspector to:
```text
http://<WIREGUARD_NODE_IP>:<NODEPORT>/mcp
```

If you use event subscriptions, run `logging/setLevel` in the client and then call the `events_subscribe` tool (the server delivers notifications via `notifications/message`).

## Configuration knobs

### Service port vs NodePort (important)
- The chart uses `.Values.service.port` for **containerPort**, **Service port**, and health probes.
- The script defaults `--app-port 8080` because this image listens on `8080` in-cluster.

If you *know* your image listens on a different internal port, override:
```bash
./mcp-k8s-up --image "$IMAGE" --node-ip <WIREGUARD_NODE_IP> --app-port 8686
```

The externally reachable port is the **NodePort** that Kubernetes assigns (printed by the script).

### RBAC
`mcp-system-rbac.yaml` grants:
- cluster-wide read via built-in `view` (broad; tighten later if desired)
- read-only access to Helm release storage (`secrets`, `configmaps`) across all namespaces
- `nodes get/list/watch` so `events_subscribe` `mode=faults` works (fault detection watches Nodes)

## Teardown
Remove the Helm release and RBAC:
```bash
./mcp-k8s-down
```

Remove everything including the namespace:
```bash
./mcp-k8s-down --delete-namespace
```

## Test: induce CrashLoopBackOff (optional)
Create a disposable workload that will CrashLoop so you can verify `faults` notifications:
```bash
kubectl delete ns mcp-test --ignore-not-found
kubectl create ns mcp-test
kubectl -n mcp-test create deploy crashy --image=busybox:1.36 -- /bin/sh -c 'sleep 1; exit 1'
```

Then subscribe with `events_subscribe` using `mode: faults` and you should receive a `CrashLoop` notification.

## Troubleshooting

- **`events_subscribe` doesn’t work in `faults` mode**: confirm Nodes RBAC:
  - `kubectl auth can-i --as=system:serviceaccount:mcp-system:kubernetes-mcp-server list nodes`
- **Pod CrashLooping immediately**: verify the internal listen port matches `--app-port` (probes hit `/healthz` on that port).
- **No notifications**: ensure the client is using streamable HTTP correctly (long-lived receive stream) and that you’ve called `logging/setLevel`.

