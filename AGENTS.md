# infra

Flux GitOps repo for the `rpi` k3s cluster. Flux reconciles `./clusters/rpi` from `main`.

## Layout

```
clusters/rpi/
├── kustomization.yaml          # cluster root
├── flux-system/                # Flux bootstrap (do not hand-edit gotk-components.yaml)
├── infrastructure/             # cluster-wide platform config
│   └── traefik/                # k3s bundled Traefik HelmChartConfig
└── apps/                       # application workloads
    ├── kustomization.yaml
    ├── image-update-automation.yaml   # one file per app, lives here not inside app dir
    └── <app>/
        ├── kustomization.yaml
        ├── workload.yaml       # Namespace + Deployment + Service
        ├── httproute.yaml      # Gateway API route
        └── source.yaml         # ImageRepository + ImagePolicy
```

## Cluster assumptions

- **Ingress:** k3s bundled Traefik in `kube-system` (do not install a separate Traefik HelmRelease).
- **Gateway API:** configured via `infrastructure/traefik/helm-chart-config.yaml`. Use `HTTPRoute` attached to Gateway `traefik-gateway` in `kube-system` (listener `web`). TLS is handled outside the cluster.
- **Cloudflare tunnel:** runs on the manager node, not in Kubernetes.
- **Domains:** public hostnames use `*.kurwidolek.com` (e.g. `breakout.kurwidolek.com`).
- **Image automation:** Flux image reflector/automation controllers are in `flux-system/gotk-image-controllers.yaml`.

## Adding a new app

Use `clusters/rpi/apps/breakout/` as the reference implementation.

### 1. Create the app directory

```
clusters/rpi/apps/<app>/
├── kustomization.yaml
├── workload.yaml
├── httproute.yaml
└── source.yaml
```

**`kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - workload.yaml
  - httproute.yaml
  - source.yaml
```

**`workload.yaml`** — single file with three documents separated by `---`:

1. `Namespace` named after the app
2. `Deployment` in that namespace
3. `ClusterIP` `Service` exposing the container port

Keep manifests minimal: no extra `securityContext` unless the image requires it.

For auto-updated container images, add the Flux setter marker on the image line:

```yaml
image: ghcr.io/bieniucieniu/<app>:latest # {"$imagepolicy": "flux-system:<app>"}
```

**`httproute.yaml`** — Gateway API route:

- `parentRefs`: Gateway `traefik-gateway` in `kube-system`, section `web`
- `hostnames`: `<app>.kurwidolek.com`
- `backendRefs`: the app Service and port

**`source.yaml`** — Flux image scanning (both resources in `flux-system` namespace):

- `ImageRepository` for `ghcr.io/bieniucieniu/<app>`
- `ImagePolicy` with `alphabetical: order: asc` when tracking `:latest`

### 2. Register the app

Add the app to `clusters/rpi/apps/kustomization.yaml`:

```yaml
resources:
  - breakout
  - <app>
  - image-update-automation.yaml   # existing shared or per-app files
```

### 3. Add image update automation

Create `clusters/rpi/apps/<app>-image-update-automation.yaml` (or extend the existing pattern) **outside** the app directory:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1
kind: ImageUpdateAutomation
metadata:
  name: <app>
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: fluxcdbot
        email: fluxcdbot@users.noreply.github.com
      messageTemplate: |
        chore(<app>): update image
        {{range .Updated.Images -}}
        - {{.}}
        {{end -}}
    push:
      branch: main
  update:
    path: ./clusters/rpi/apps/<app>
    strategy: Setters
```

Reference it from `clusters/rpi/apps/kustomization.yaml`.

The `flux-system` GitRepository credentials must allow push to `main` for image bumps to commit back.

### 4. Wire up external access

In Cloudflare (on the manager tunnel), route `<app>.kurwidolek.com` to k3s Traefik over HTTP (port 80).

## k3s Traefik

Traefik is installed by k3s. Its Gateway API settings are managed in GitOps:

```
clusters/rpi/infrastructure/traefik/helm-chart-config.yaml
```

Do **not** keep a duplicate at `/mnt/sda1/k3s/server/manifests/traefik-gateway.yaml` — delete the server file after Flux owns the config.

Do **not** create a separate `Gateway` resource — k3s Helm creates `traefik-gateway` automatically.

The `namespacePolicy.from: All` setting is required so HTTPRoutes in app namespaces (e.g. `breakout`) are accepted.

After changing the HelmChartConfig, verify Traefik picked it up:

```bash
kubectl get helmchartconfig traefik -n kube-system -o yaml
kubectl get gateway traefik-gateway -n kube-system \
  -o jsonpath='{.spec.listeners[0].allowedRoutes.namespaces.from}{"\n"}'
```

## Conventions

| Item | Convention |
|------|------------|
| App namespace | same as app name (`breakout`, `myapp`, …) |
| Labels | `app.kubernetes.io/name`, `app.kubernetes.io/part-of` |
| Flux image resources | `metadata.namespace: flux-system`, name matches app |
| HTTPRoute parent | `traefik-gateway` / `kube-system` / listener `web` |
| Hostname | `<app>.kurwidolek.com` |

## Validate locally

```bash
kubectl kustomize clusters/rpi/apps/<app>
kubectl kustomize clusters/rpi
```
