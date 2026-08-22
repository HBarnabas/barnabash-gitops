# pc-k8s storage note

The app manifests in this repo request the `local-path` StorageClass so they
work unchanged on both k3s (built-in) and full k8s.

On a vanilla k8s cluster you need to install a provisioner that creates that
StorageClass. Pick one:

## Option A — Rancher local-path-provisioner (simplest, single-node friendly)

```sh
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

To manage it via GitOps instead, add an `OCIRepository`/`GitRepository` source
pointing at the manifest and a `Kustomization` here.

## Option B — Longhorn / OpenEBS (multi-node, replicated volumes)

Add a `HelmRepository` + `HelmRelease` in this directory. Then either keep the
`local-path` name via a StorageClass alias, or override `storageClassName` in
the `apps/pc-k8s` overlay patches.
