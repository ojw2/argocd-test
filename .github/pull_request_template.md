## What changed

<!-- Describe the change -->

## Path affected

- [ ] gitops/platform
- [ ] gitops/infrastructure
- [ ] gitops/workloads
- [ ] gitops/applications

## Checklist

- [ ] Manifests validated locally (`kubectl kustomize` or `helm template` ran clean)
- [ ] No plaintext secrets committed
- [ ] Resource requests/limits set on any new containers
- [ ] Network policies updated if new service-to-service traffic is introduced
- [ ] Commit is signed (`git commit -S`)

## Rollback plan

<!-- How to revert if ArgoCD sync causes issues -->
