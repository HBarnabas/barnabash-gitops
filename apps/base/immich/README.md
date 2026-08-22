# Immich

Immich needs **PostgreSQL with a vector extension** (VectorChord, formerly
pgvecto-rs). The Helm chart does not deploy Postgres for you.

## 1. Provide the database

Pick one and deploy it into the `immich` namespace as `immich-postgres`
(the hostname referenced in `helmrelease.yaml`):

- **Quick / single-node:** run the `ghcr.io/tensorchord/pgvecto-rs` (or the
  Immich-recommended `ghcr.io/immich-app/postgres`) image as a `StatefulSet`
  with a PVC.
- **Recommended / production:** deploy the CloudNativePG operator and a
  `Cluster` using the pgvecto-rs image. This gives you backups + PITR.

Whichever you choose, create a role/db named `immich` and put its password in
`secret.sops.yaml`.

## 2. Set the secret

```sh
sops apps/base/immich/secret.sops.yaml   # edit, set the password
```

## 3. Set your domain + TLS

Edit `helmrelease.yaml`: replace `immich.example.com`, and once the staging
certificate works switch the annotation to `letsencrypt-prod`.

## Storage

The `immich-library` PVC uses `local-path`. For large libraries point it at a
dedicated disk (bigger PVC on pc-k8s is already patched in `apps/pc-k8s`), or
switch to an NFS/hostPath-backed StorageClass mapped to your media drive.
