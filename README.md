# Uptime Kuma Helm Chart

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

## Installation

### 1. Publish and install the chart

Helm supports two publishing models: **OCI registries** (the modern path) and **classic HTTP chart repositories**. Both are shown below.

Key values:

| Value | Default | Description |
|---|---|---|
| `image.tag` | `"2"` | Uptime Kuma image tag |
| `httpRoute.parentRefs` | `traefik-gateway / ingress` | Gateway the HTTPRoute attaches to |
| `httpRoute.hostnames` | `[]` | Hostnames to match (empty = all) |
| `persistence.storageClass` | `ceph-rbd` | StorageClass for the data PVC |
| `persistence.size` | `1Gi` | PVC size |
| `fullnameOverride` | `""` | Pin all resource names (e.g. `uptime-kuma`) |

The install commands below omit `--set` flags for brevity. Pass the same values either as `--set` flags or with a values file:

```bash
# Inline
helm upgrade --install uptime-kuma <chart-ref> --set 'httpRoute.hostnames[0]=uptime.example.com'

# Values file (preferred when overriding multiple values)
helm upgrade --install uptime-kuma <chart-ref> -f my-values.yaml
```

#### OCI registry (recommended)

OCI lets you push charts to any container registry, including the MicroK8s built-in registry.

##### MicroK8s built-in registry

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

##### GitHub Container Registry (GHCR)

**Using the `gh` CLI** (recommended — uses credentials from `gh auth login`, no token management needed):

```sh
gh auth status
```

`gh auth login` does not request `write:packages` by default. Add it once before pushing:

```bash
gh auth refresh -s write:packages
```

```sh
helm package charts/uptime-kuma
```

```bash
cat ~/.config/helm/registry/config.json
# if GHCR is not in the auths section:
gh auth token | helm registry login ghcr.io --username <github-user> --password-stdin

helm push uptime-kuma-*.tgz oci://ghcr.io/<github-user>/charts
```

GHCR defaults new packages to private. `helm push` uses the OCI protocol which has no visibility concept, so there is no way to set it at push time. Make the package public once after the first push — it stays public for all subsequent pushes to the same package. Go to **github.com → your profile → Packages → charts/uptime-kuma → Package settings → Change visibility → Public**.

View published versions:

```sh
gh api /user/packages/container/charts%2Fuptime-kuma/versions --jq '.[].metadata.container.tags'
```

**Using a personal access token (PAT):** Create one at **GitHub → Settings → Developer settings → Personal access tokens** with `write:packages` scope, then set it in your shell:

```bash
export GITHUB_TOKEN=ghp_...
```

```bash
echo $GITHUB_TOKEN | helm registry login ghcr.io --username <github-user> --password-stdin

helm push uptime-kuma-*.tgz oci://ghcr.io/<github-user>/charts
```

Install from the registry:

```bash
helm upgrade --install uptime-kuma oci://ghcr.io/<github-user>/charts/uptime-kuma \
  --version 0.1.0 \
  --namespace uptime-kuma --create-namespace
```

#### Classic HTTP repository (GitHub Pages)

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

**Local network access by hostname:** Add a corresponding entry to `/etc/hosts` on any machine that needs to reach the web UI. Any node IP works since Traefik runs as a DaemonSet on every node:

```
192.168.1.100  uptime.home
```

### Versioning

Bump `version` in `charts/uptime-kuma/Chart.yaml` before every publish. `appVersion` tracks the upstream Uptime Kuma release and is independent of the chart version.

## Configuring monitor alerts

For an easy and free public notification service with no account needed, use ntfy.

Generate a random topic name. Use your service name to make it easy to recognize.
```sh
echo myservice-$(openssl rand -hex 4)
```

In Uptime Kuma:

1. Select your monitor, *Edit*.
2. In the Notifications section, *Set Up Notification*.
3. Notification Type: In the Push Services section, select **Ntfy**.
4. Give it a *Friendly Name* and paste your generated **ntfy Topic**.
