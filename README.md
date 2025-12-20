# Deploy `kubernetes-mcp-server-with-events` on a k0s cluster

This repo contains a minimal, repeatable way to deploy **Randy’s fork** of `kubernetes-mcp-server` (the image `ghcr.io/randybias/kubernetes-mcp-server-with-events`) into a Kubernetes cluster (k0s tested) using the upstream Helm chart `oci://ghcr.io/containers/charts/kubernetes-mcp-server`.

Important: the **event subscription/notification tooling** (`events_subscribe`, `mode: faults`, `notifications/message`, etc.) is provided by this forked image. If you deploy the upstream image instead, those event tools will not be present.

It creates:
- Namespace: `mcp-system`
- ServiceAccount + RBAC for in-cluster auth
- A `NodePort` Service so you can reach it from a laptop over VPN/WireGuard

No Istio, no ingress, no auth gateway.

## Contents
- `mcp-k8s-up`: idempotent setup (RBAC + Helm install/upgrade)
- `mcp-k8s-down`: idempotent teardown (Helm uninstall + RBAC cleanup)
- `mcp-system-rbac.yaml`: the manifests applied by the scripts

## Prerequisites (run from the k0s node or any admin host)
- `kubectl` configured to talk to the cluster (admin creds)
- `helm` installed
- Cluster nodes can pull images from GHCR

## Deploy

1) Choose the image
```bash
IMAGE=ghcr.io/randybias/kubernetes-mcp-server-with-events:dev-latest
```

2) Run setup
```bash
./mcp-k8s-up --image "$IMAGE" --node-ip <WIREGUARD_NODE_IP>
```

What it does:
- `kubectl apply -f mcp-system-rbac.yaml`
- `helm upgrade --install kubernetes-mcp-server oci://ghcr.io/containers/charts/kubernetes-mcp-server -n mcp-system ...`
- sets `service.type=NodePort` (Kubernetes auto-assigns a random nodePort)
- prints:
  - the assigned NodePort
  - a laptop URL like `http://<WIREGUARD_NODE_IP>:<NODEPORT>`

3) Connect from your laptop
- MCP base URL: `http://<WIREGUARD_NODE_IP>:<NODEPORT>/mcp`
- Health check: `http://<WIREGUARD_NODE_IP>:<NODEPORT>/healthz`

## Important knobs

### `--app-port` (internal server port)
The upstream chart uses `.Values.service.port` for:
- container `containerPort`
- Service port
- liveness/readiness probes (to `/healthz`)

This repo defaults to `--app-port 8080` because the `kubernetes-mcp-server-with-events` image listens on `8080` in-cluster.

If your image listens on a different port:
```bash
./mcp-k8s-up --image "$IMAGE" --node-ip <WIREGUARD_NODE_IP> --app-port 8686
```

The laptop-facing port remains the Service **NodePort** (auto-assigned and printed).

### RBAC (`mcp-system-rbac.yaml`)
This grants (broad for now):
- `ClusterRoleBinding` to built-in `view` (cluster-wide read)
- Helm release storage read: `secrets` and `configmaps` (`get/list/watch`) across all namespaces
- Nodes read (`nodes get/list/watch`) required for this fork’s `events_subscribe` with `mode: faults`

## Tear down
Remove the Helm release and RBAC objects:
```bash
./mcp-k8s-down
```

Remove everything including the namespace:
```bash
./mcp-k8s-down --delete-namespace
```

## Test: trigger a CrashLoopBackOff notification

This test requires the forked image (`ghcr.io/randybias/kubernetes-mcp-server-with-events:*`) because upstream `kubernetes-mcp-server` does not include the event subscription tools.

1) Subscribe in your MCP client:
- Call `logging/setLevel` (e.g., `debug` or `warning`)
- Call tool `events_subscribe` with `{"mode":"faults"}`

2) Create a crashing workload:
```bash
kubectl delete ns mcp-test --ignore-not-found
kubectl create ns mcp-test
kubectl -n mcp-test create deploy crashy --image=busybox:1.36 -- /bin/sh -c 'sleep 1; exit 1'
```

You should receive a `notifications/message` with `faultType: CrashLoop`.

## Troubleshooting

- `events_subscribe` fails in `faults` mode: verify Nodes RBAC:
  - `kubectl auth can-i --as=system:serviceaccount:mcp-system:kubernetes-mcp-server list nodes`
- Pod is CrashLooping: your internal server port doesn’t match `--app-port` (probes will fail).
- You can reach `/healthz` but not MCP: make sure you’re using `/mcp` and the client keeps the receive stream open (streamable HTTP).
