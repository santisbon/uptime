# uptime

Helm chart for [Uptime Kuma](https://github.com/louislam/uptime-kuma), a self-hosted monitoring tool, targeting a MicroK8s cluster with the Ceph RBD storage class and Gateway API HTTPRoute ingress.

## Prerequisites

- MicroK8s with addons: `ingress`, `rook-ceph`, `registry`, `dns`, `helm`, `helm3`
- MicroCeph with RGW enabled and the RBD kernel module loaded
- `gateway.networking.k8s.io` CRDs (installed by the MicroK8s ingress addon)
- Helm 3

## Install

Install directly from this repository when you have the source checked out locally and are deploying to a single cluster you manage yourself. This is the simplest path: no packaging or publishing step required, and changes to the chart take effect on the next `helm upgrade --install`.

```bash
helm upgrade --install uptime-kuma ./charts/uptime-kuma \
  --namespace uptime-kuma --create-namespace \
  --set 'httpRoute.hostnames[0]=uptime.example.com'
```

Publish the chart to a registry or repository (see [Publishing the chart](#publishing-the-chart)) when you need to share it across multiple clusters, teams, or CI/CD pipelines, or when you want to pin deployments to a specific released version rather than whatever is currently on disk.

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

### OCI registry (recommended)

OCI lets you push charts to any container registry, including the MicroK8s built-in registry.

#### MicroK8s built-in registry

The MicroK8s registry addon runs an unauthenticated registry at `localhost:32000` on every node.

```bash
# Package the chart
helm package charts/uptime-kuma

# Push (Helm 3.8+)
helm push uptime-kuma-*.tgz oci://localhost:32000/charts
```

View published charts:

```bash
# List all repositories in the registry
curl -s http://localhost:32000/v2/_catalog | jq

# List available versions of the chart
curl -s http://localhost:32000/v2/charts/uptime-kuma/tags/list | jq

# Inspect chart metadata for a specific version
helm show chart oci://localhost:32000/charts/uptime-kuma --version 0.1.0
```

Install directly from it:

```bash
helm upgrade --install uptime-kuma oci://localhost:32000/charts/uptime-kuma --version 0.1.0 \
  --namespace uptime-kuma --create-namespace
```

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
