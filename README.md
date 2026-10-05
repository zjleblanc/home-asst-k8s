# Home Assistant on Kubernetes (ArgoCD)

Deploy [Home Assistant](https://www.home-assistant.io/) on [MicroShift](https://microshift.io/)/OpenShift, managed by [ArgoCD](https://argo-cd.readthedocs.io/).

- **Ingress**: OpenShift `Route` (via the HAProxy router), not NGINX Ingress
- **TLS**: cert-manager, auto-injected into the Route via the [openshift-routes](https://github.com/cert-manager/openshift-routes) integration
- **Secrets**: [External Secrets Operator](https://external-secrets.io/) pulling from Vault, rendered as HA's `secrets.yaml`

## Repository Layout

```
├── base/                        # Shared manifests
│   ├── kustomization.yaml
│   ├── namespace.yaml           # home-assistant namespace
│   ├── serviceaccount.yaml      # Dedicated SA for the HA pod
│   ├── scc-rolebinding.yaml     # Grants the SA the anyuid SCC (MicroShift/OpenShift)
│   ├── externalsecret.yaml      # ESO ExternalSecret, synced from Vault
│   ├── deployment.yaml          # HA container + probes + resources + secrets.yaml mount
│   ├── service.yaml             # ClusterIP on port 8123
│   ├── pvc.yaml                 # 5 Gi config volume
│   └── route.yaml               # OpenShift Route w/ cert-manager annotation
└── overlays/
    └── default/                 # Default environment overlay
        ├── kustomization.yaml
        └── patches/
            ├── route-host.yaml      # Sets hostname to ha.leblanc.rodeo
            └── deployment-tz.yaml   # Set your timezone
```

## Quick Start

### 1. Customize the overlay

The default overlay already sets the hostname to `ha.leblanc.rodeo`. Adjust these patches if needed:

| Patch | What it does |
|---|---|
| `route-host.yaml` | Sets `spec.host` on the Route — currently `ha.leblanc.rodeo` |
| `deployment-tz.yaml` | Sets the `TZ` env var — currently `America/Chicago`, see [TZ identifiers](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) |

### 2. Preview the rendered manifests

```bash
kubectl kustomize overlays/default
```

### 3. Create the ArgoCD Application

Apply the following manifest to your cluster (adjust `repoURL`, `targetRevision`, and `server` as needed):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: home-assistant
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/zjleblanc/home-asst-k8s.git
    targetRevision: main
    path: overlays/default
  destination:
    server: https://kubernetes.default.svc
    namespace: home-assistant
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f argocd-application.yaml
```

> **Tip:** You can also create the app via the ArgoCD CLI:
> ```bash
> argocd app create home-assistant \
>   --repo https://github.com/zjleblanc/home-asst-k8s.git \
>   --path overlays/default \
>   --dest-server https://kubernetes.default.svc \
>   --dest-namespace home-assistant \
>   --sync-policy automated \
>   --auto-prune \
>   --self-heal
> ```

## Secrets Management (External Secrets Operator + Vault)

`base/externalsecret.yaml` defines an `ExternalSecret` that pulls everything under a Vault KV path via `dataFrom.extract`, then templates it into a single `secrets.yaml` key on the resulting Kubernetes `Secret` (`home-assistant-secrets`). That secret is mounted read-only at `/config/secrets.yaml` in the Deployment, so any key you store in Vault becomes usable in `configuration.yaml` via HA's native [`!secret`](https://www.home-assistant.io/docs/configuration/secrets/) syntax:

```yaml
# configuration.yaml
mqtt:
  broker: !secret mqtt_broker
  username: !secret mqtt_username
  password: !secret mqtt_password
```

As long as `mqtt_broker`, `mqtt_username`, and `mqtt_password` exist as keys at the Vault path configured in `dataFrom[0].extract.key`, ESO will sync them automatically on the `refreshInterval` (1h by default).

**Before this works**, confirm/update the two TODOs in `base/externalsecret.yaml`:
- `secretStoreRef.name` — must match an existing `ClusterSecretStore` (or switch `kind: SecretStore` if it's namespace-scoped)
- `dataFrom[0].extract.key` — the Vault KV v2 path (e.g. `secret/data/home-assistant`)

To add env-var-style secrets instead of file-based ones, add a second `ExternalSecret` with discrete `data` entries and reference it from the Deployment via `envFrom.secretRef`.

## TLS / Routing (OpenShift Route + cert-manager)

`base/route.yaml` is a standard OpenShift `Route` with `termination: edge` and `insecureEdgeTerminationPolicy: Redirect` — HTTP requests are redirected to HTTPS, and TLS terminates at the router (backend traffic to the pod stays plain HTTP on port 8123).

The `cert-manager.io/cluster-issuer` annotation assumes the [cert-manager-openshift-routes](https://github.com/cert-manager/openshift-routes) controller is already running in-cluster. That controller watches annotated Routes, provisions a `Certificate` via cert-manager, and writes the resulting cert/key directly into `spec.tls.certificate` / `spec.tls.key` on the Route (OpenShift Routes don't support referencing a Secret by name the way Ingress does — the cert must be inlined).

Update the `letsencrypt-prod` placeholder in `base/route.yaml` to match your actual `ClusterIssuer` name.

## MicroShift / OpenShift SCC

The upstream Home Assistant image runs as **root** by default (its `s6-overlay` init system expects it, and some integrations need it for device access). MicroShift's default `restricted`/`restricted-v2` SCC rejects that, so the Deployment runs under a dedicated ServiceAccount instead of `default`:

- `base/serviceaccount.yaml` — a `home-assistant` ServiceAccount, referenced by the Deployment via `spec.template.spec.serviceAccountName`
- `base/scc-rolebinding.yaml` — a namespace-scoped `RoleBinding` granting that ServiceAccount the built-in `anyuid` SCC (via the auto-generated `system:openshift:scc:anyuid` ClusterRole)

`anyuid` permits the container to run as any UID — including root — without granting the broader privileges of the `privileged` SCC (host networking, host paths, extra Linux capabilities, etc. all remain denied). This is scoped narrowly: only the `home-assistant` ServiceAccount in this namespace gets it, not the whole namespace or cluster.

**If you enable the optional patches below**, note they need *additional* SCC grants beyond `anyuid`:
- **Device access** (hostPath volumes) → bind the ServiceAccount to `hostmount-anyuid` instead (or additionally), since `anyuid` alone doesn't permit the host directory volume plugin.
- **Host network mode** → bind the ServiceAccount to the `hostnetwork` SCC (or `hostnetwork-v2`), since `anyuid` doesn't permit `hostNetwork: true`.

Add a second `RoleBinding` (copy `scc-rolebinding.yaml`, swap the `roleRef.name`) in an overlay patch/resource rather than widening the base grant, so environments that don't need device/host-network access stay on the narrower `anyuid` SCC.

## Customization

### Adding a new overlay

Copy the `overlays/default` directory to create environment-specific variants:

```bash
cp -r overlays/default overlays/prod
```

Then edit the patches and point your ArgoCD Application `path` to `overlays/prod`.

### Storage

The base PVC requests **5 Gi** with `ReadWriteOnce` access. To change the size or storage class, add a patch in your overlay:

```yaml
# overlays/prod/patches/pvc-size.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: home-assistant-config
spec:
  resources:
    requests:
      storage: 20Gi
  storageClassName: my-storage-class
```

### Device access (Zigbee / Z-Wave / Bluetooth)

If you need USB device passthrough for Zigbee/Z-Wave sticks, add a patch that sets `privileged: true` and mounts the device:

```yaml
# overlays/prod/patches/device-access.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: home-assistant
spec:
  template:
    spec:
      containers:
        - name: home-assistant
          securityContext:
            privileged: true
          volumeMounts:
            - name: usb-device
              mountPath: /dev/ttyUSB0
      volumes:
        - name: usb-device
          hostPath:
            path: /dev/ttyUSB0
```

### Host network mode

Some integrations (mDNS discovery, Homekit, etc.) work best with host networking:

```yaml
# overlays/prod/patches/host-network.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: home-assistant
spec:
  template:
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
```

## Image Updates

To pin a specific Home Assistant version, add an image override in your overlay's `kustomization.yaml`:

```yaml
images:
  - name: ghcr.io/home-assistant/home-assistant
    newTag: "2026.10.0"
```
