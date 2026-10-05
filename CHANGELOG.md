# Changelog

## 2026-10-05 — Scaffold Home Assistant Kustomize deployment for MicroShift/ArgoCD

### Added
- Kustomize `base/` + `overlays/default/` structure for deploying Home Assistant on Kubernetes
- `Deployment`, `Service`, `PersistentVolumeClaim`, and `Namespace` manifests for Home Assistant core
- OpenShift `Route` (in place of an Ingress) with a `cert-manager.io/cluster-issuer` annotation for automatic TLS via the [cert-manager openshift-routes](https://github.com/cert-manager/openshift-routes) integration
- `ExternalSecret` resource syncing Vault-backed secrets via External Secrets Operator, rendered into Home Assistant's `secrets.yaml` and mounted read-only into the pod
- Dedicated `ServiceAccount` + namespace-scoped `RoleBinding` granting the `anyuid` SCC, so the (root-requiring) upstream HA image runs under MicroShift's default `restricted` SCC
- Default overlay patches for hostname (`ha.leblanc.rodeo`) and timezone (`America/Chicago`)
- `README.md` covering quick start, the ArgoCD `Application` manifest, secrets management, TLS/routing, the MicroShift SCC setup, and customization recipes (storage sizing, device access, host networking, image pinning)
