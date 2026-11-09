# fluxcon-d2-fleet

Fleet repository for the FluxCon 2026 demo *"Your VMs Are Just CRDs"*: the D2
**fleet** analogue, reduced to one cluster (`clusters/dev`).

- `clusters/dev/flux-system/flux-instance.yaml`: the Flux Operator `FluxInstance`.
  Its root sync is this repository's signed OCI artifact
  (`oci://ghcr.io/wrkode/fluxcon-d2-fleet:latest`), and a kustomize patch adds a
  Cosign keyless `verify` policy to the generated root `OCIRepository`.
- `tenants/infra.yaml`: a `ResourceSet` generating the signed sources and
  Kustomizations `kubeswift-platform` → `kubeswift-infra` from
  [fluxcon-d2-infra](https://github.com/wrkode/fluxcon-d2-infra).
- `tenants/vms.yaml`: a `ResourceSet` creating the `workloads` tenant and the
  `kubeswift-workloads` Kustomization (depends on `kubeswift-infra`) from
  [fluxcon-d2-vms](https://github.com/wrkode/fluxcon-d2-vms), impersonating a
  service account that can only manage `SwiftGuestPool`s in `workloads`.

```
Git push ─▶ GitHub Actions: validate ▸ flux push artifact ▸ cosign sign (OIDC) ▸ cosign verify ▸ tag latest
                                   │
                                   ▼
           ghcr.io/wrkode/fluxcon-d2-{fleet, infra/kubeswift-platform, infra/kubeswift-infra, vms/web-fleet}
                                   │  (signed Kubernetes config)
                                   ▼
FluxInstance ─▶ OCIRepository flux-system (verify) ─▶ Kustomization tenants ─▶ ResourceSets infra, vms
   ResourceSet infra ─▶ OCIRepository + Kustomization kubeswift-platform ─▶ HelmRelease kubeswift
                     ─▶ OCIRepository + Kustomization kubeswift-infra    ─▶ SwiftGuestClass, SwiftImage, SwiftSeedProfile
   ResourceSet vms   ─▶ OCIRepository + Kustomization kubeswift-workloads ─▶ SwiftGuestPool/web ─▶ VMs

ghcr.io/wrkode/fluxcon-d2-infra/vm-images:web-v{1,2}  (VM disks, a different OCI content type)
        └──▶ SwiftImage.spec.source.oci (pinned by digest) ─▶ imported root disk ─▶ guest clones
```

Differences from upstream D2: one cluster; no Flux Operator self-management
(the operator is a separate Helm release); single-tenant controllers
(`multitenant: false`) with an explicit impersonated service account for the
VMs tenant only; component artifacts are signed and promoted to `latest` by
plain `flux push artifact` / `cosign` / `flux tag artifact` steps instead of the
D2 push action.

Based on the Flux D2 reference architecture by ControlPlane
([d2-fleet](https://github.com/controlplaneio-fluxcd/d2-fleet), Apache-2.0).
