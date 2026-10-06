# Changelog

## 2026-10-05 — Add custom SCC for Home Assistant host-network deployment

### Added
- Created `overlays/default/scc-anyuid-hostnetwork.yaml`, a custom `SecurityContextConstraints` that combines `anyuid` (arbitrary UID for root access) and `hostnetwork` capabilities, required because OpenShift admission only selects a single SCC per pod.
- Created `overlays/default/sa-rolebinding-scc.yaml` to grant the custom SCC to the `home-assistant` ServiceAccount.

### Changed
- Refined `overlays/default/kustomization.yaml` to remove the blanket `namespace` transformer, preventing it from incorrectly stamping a namespace on the cluster-scoped SCC; the RoleBinding now sets its own namespace explicitly.

### Removed
- Deleted `overlays/default/scc-rolebinding-hostnetwork.yaml`, as it is superseded by the custom SCC that allows both root access and host networking.

## 2026-10-05 — Fix Home Assistant CrashLoopBackOff from SCC RoleBinding regression

### Fixed
- Restored `base/scc-rolebinding.yaml` to grant the `anyuid` SCC — it had been overwritten to grant `hostnetwork-v2` instead, which forces a random non-root UID and breaks the HA container's s6-overlay `/run` setup (`wrong permissions on /run for a gid 0 setup`)
- Added a separate `overlays/default/scc-rolebinding-hostnetwork.yaml` `RoleBinding` granting `hostnetwork-v2`, so host-network overlays get both SCCs without widening the base grant
- Set `namespace: home-assistant` on the overlay `Kustomization` so the new RoleBinding resource namespaces correctly

## 2026-10-05 — Scaffold Home Assistant Kustomize deployment for MicroShift/ArgoCD

### Added
- Kustomize `base/` + `overlays/default/` structure for deploying Home Assistant on Kubernetes
- `Deployment`, `Service`, `PersistentVolumeClaim`, and `Namespace` manifests for Home Assistant core
- OpenShift `Route` (in place of an Ingress) with a `cert-manager.io/cluster-issuer` annotation for automatic TLS via the [cert-manager openshift-routes](https://github.com/cert-manager/openshift-routes) integration
- `ExternalSecret` resource syncing Vault-backed secrets via External Secrets Operator, rendered into Home Assistant's `secrets.yaml` and mounted read-only into the pod
- Dedicated `ServiceAccount` + namespace-scoped `RoleBinding` granting the `anyuid` SCC, so the (root-requiring) upstream HA image runs under MicroShift's default `restricted` SCC
- Default overlay patches for hostname (`ha.leblanc.rodeo`) and timezone (`America/Chicago`)
- `README.md` covering quick start, the ArgoCD `Application` manifest, secrets management, TLS/routing, the MicroShift SCC setup, and customization recipes (storage sizing, device access, host networking, image pinning)
