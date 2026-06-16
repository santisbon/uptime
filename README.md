# uptime

Helm chart for [Uptime Kuma](https://github.com/louislam/uptime-kuma), a self-hosted monitoring tool, targeting a MicroK8s cluster with the Ceph RBD storage class and Gateway API HTTPRoute ingress.

## Prerequisites

- Helm 3
- A Kubernetes cluster with `gateway.networking.k8s.io` CRDs and a provisioned Gateway (the MicroK8s `ingress` addon satisfies both, providing a `traefik-gateway` Gateway in the `ingress` namespace)
- A StorageClass for the data PVC (defaults to `ceph-rbd`; set `persistence.storageClass` to use a different one)
- A kubeconfig pointing at the cluster. If you're running Helm from a machine that is not a cluster node, copy the kubeconfig from any node and replace the loopback address with the node's LAN IP or host name. If your cluster node user is `ubuntu` and a node is `node-01.local`:

  ```bash
  ssh ubuntu@node-01.local "microk8s config" \
    | sed 's/127.0.0.1/node-01.local/' \
    > ~/.kube/microk8s.yaml
  export KUBECONFIG=~/.kube/microk8s.yaml
  ```

  To avoid setting `KUBECONFIG` in every shell session, add the export to your `~/.bashrc` or `~/.zshrc`, or merge it into your existing `~/.kube/config`:

  ```bash
  KUBECONFIG=~/.kube/config:~/.kube/microk8s.yaml \
    kubectl config view --flatten > ~/.kube/config
  ```

## Install

**Install directly from this repository** when you have the source checked out locally and are deploying to a single cluster you manage yourself. This is the simplest path: no packaging or publishing step required, and changes to the chart take effect on the next `helm upgrade --install`.

```bash
helm upgrade --install uptime-kuma ./charts/uptime-kuma \
  --namespace uptime-kuma --create-namespace \
  --set 'httpRoute.hostnames[0]=uptime.example.com'
```

**Publish the chart to a registry or repository** (see [Publishing the chart](#publishing-the-chart)) when you need to share it across multiple clusters, teams, or CI/CD pipelines, or when you want to pin deployments to a specific released version rather than whatever is currently on disk.

**Local network access by hostname:** Set a hostname and add a corresponding entry to `/etc/hosts` on any machine that needs to reach it:

```bash
helm upgrade --install uptime-kuma ./charts/uptime-kuma \
  --namespace uptime-kuma --create-namespace \
  --set 'httpRoute.hostnames[0]=uptime.home'
```

```
# /etc/hosts
192.168.1.100  uptime.home
```

This is the right approach when running multiple apps on the same cluster — each app gets its own hostname pointing to the same IP, and Traefik reads the `Host` header to route each request to the correct service. Any node's IP works for the `/etc/hosts` entry since Traefik runs as a DaemonSet on every node.

If you only have one app and want the simplest possible setup, omit `httpRoute.hostnames` entirely and access Uptime Kuma directly by any node's LAN IP address (e.g. `http://192.168.1.100`). No hostname or `/etc/hosts` entry needed.

**Exposing to the internet without port forwarding:** Use [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) (`cloudflared`), which opens an outbound-only connection from your network to Cloudflare's edge. No open inbound ports or router configuration needed. It's free, works with a custom domain you manage in Cloudflare, and handles HTTPS automatically. Run `cloudflared` as a service on the MicroK8s node pointing at the node's LAN IP and port, and set `httpRoute.hostnames` to your Cloudflare-managed domain. As an alternative, [ngrok](https://ngrok.com) is simpler to set up but the free tier assigns a random URL that changes every time the agent restarts.

Key values:

| Value | Default | Description |
|---|---|---|
| `image.tag` | `"2"` | Uptime Kuma image tag |
| `httpRoute.parentRefs` | `traefik-gateway / ingress` | Gateway the HTTPRoute attaches to |
| `httpRoute.hostnames` | `[]` | Hostnames to match (empty = all) |
| `persistence.storageClass` | `ceph-rbd` | StorageClass for the data PVC |
| `persistence.size` | `1Gi` | PVC size |
| `fullnameOverride` | `""` | Pin all resource names (e.g. `uptime-kuma`) |

## Publishing the chart

Helm supports two publishing models: **OCI registries** (the modern path) and **classic HTTP chart repositories**. Both are shown below.

The install commands in this section omit `--set` flags for brevity. Pass the same values you would when installing from source, either as `--set` flags or with a values file:

```bash
# Inline
helm upgrade --install uptime-kuma <chart-ref> --set 'httpRoute.hostnames[0]=uptime.example.com'

# Values file (preferred when overriding multiple values)
helm upgrade --install uptime-kuma <chart-ref> -f my-values.yaml
```

### OCI registry (recommended)

OCI lets you push charts to any container registry, including the MicroK8s built-in registry.

#### MicroK8s built-in registry

The MicroK8s registry addon exposes an unauthenticated registry on port 32000 on every node. Use any node's LAN IP or host name to reach it from your laptop.

```bash
# Package the chart
helm package charts/uptime-kuma

# Push (Helm 3.8+)
helm push uptime-kuma-*.tgz oci://node-01.local:32000/charts --plain-http # if you don't have https
```

View published charts:

```bash
# List all repositories in the registry
curl -s http://node-01.local:32000/v2/_catalog | jq

# List available versions of the chart
curl -s http://node-01.local:32000/v2/charts/uptime-kuma/tags/list | jq

# Inspect chart metadata for a specific version
helm show chart oci://node-01.local:32000/charts/uptime-kuma --version 0.1.0 --plain-http # if you don't have https
```

Install directly from it:

*Assumes you've set up your `/etc/hosts` for `uptime.home` with a node's IP.*
```bash
helm upgrade --install uptime-kuma oci://node-01.local:32000/charts/uptime-kuma --version 0.1.0 \
  --namespace uptime-kuma --create-namespace --set 'httpRoute.hostnames[0]=uptime.home' \
  --plain-http # if you don't have https
```

```sh
helm list --all-namespaces
kubectl get pvc -n uptime-kuma # Should be Bound
kubectl get pods -n uptime-kuma # Should be Running
```

Go to any node's IP/hostname e.g. http://uptime.home

#### GitHub Container Registry (GHCR)

```bash
helm package charts/uptime-kuma

echo $GITHUB_TOKEN | helm registry login ghcr.io --username <github-user> --password-stdin

helm push uptime-kuma-*.tgz oci://ghcr.io/<github-user>/charts
```

View published charts:

```bash
# List available versions via the GitHub API
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/user/packages/container/charts%2Fuptime-kuma/versions" \
  | jq '.[].metadata.container.tags'

# Inspect chart metadata for a specific version
helm show chart oci://ghcr.io/<github-user>/charts/uptime-kuma --version 0.1.0
```

Install:

```bash
helm upgrade --install uptime-kuma oci://ghcr.io/<github-user>/charts/uptime-kuma --version 0.1.0 \
  --namespace uptime-kuma --create-namespace
```

### Classic HTTP chart repository (GitHub Pages)

A classic repo is a static directory containing packaged `.tgz` files and an `index.yaml` manifest, served over HTTP.

1. **Package the chart:**

   ```bash
   helm package charts/uptime-kuma --destination .deploy/
   ```

2. **Generate or update the index:**

   ```bash
   # First publish: build the index from scratch
   helm repo index .deploy/ --url https://<github-user>.github.io/<repo>

   # Subsequent publishes: merge into an existing hosted index
   helm repo index .deploy/ --url https://<github-user>.github.io/<repo> \
     --merge <(curl -s https://<github-user>.github.io/<repo>/index.yaml)
   ```

3. **Publish** the contents of `.deploy/` to the `gh-pages` branch (or whichever branch GitHub Pages serves from).

4. **Add the repo and install:**

   ```bash
   helm repo add uptime https://<github-user>.github.io/<repo>
   helm repo update
   helm upgrade --install uptime-kuma uptime/uptime-kuma \
     --namespace uptime-kuma --create-namespace
   ```

5. **View published charts:**

   ```bash
   helm search repo uptime
   ```

   Or inspect the raw index directly:

   ```bash
   curl -s https://<github-user>.github.io/<repo>/index.yaml
   ```

### Versioning

Bump `version` in `charts/uptime-kuma/Chart.yaml` before every publish. `appVersion` tracks the upstream Uptime Kuma release and is independent of the chart version.
