# Woow_k3s_immich — Immich Helm Chart for K3s/Kubernetes

[繁體中文](README_zh-TW.md)

Helm chart deploying [Immich](https://immich.app) — a self-hosted photo and
video management platform — on K3s/Kubernetes. Bundles the Immich server, the
machine-learning worker (face recognition, smart search, auto-tagging), an
in-cluster Redis, and a `pgvecto.rs` PostgreSQL StatefulSet.

> **Looking for another platform?**
> Docker/Podman Compose → [Woow_podman_immich](https://github.com/WOOWTECH/Woow_podman_immich) ·
> Home Assistant add-on → [Woow_ha_immich](https://github.com/WOOWTECH/Woow_ha_immich)

## Architecture

| Component | Image | Service | NodePort |
|---|---|---|---|
| immich-server | `ghcr.io/immich-app/immich-server:release` | `immich-server:2283` | `30283` |
| immich-machine-learning | `ghcr.io/immich-app/immich-machine-learning:release` | `immich-machine-learning:3003` | — |
| postgres (pgvecto.rs) | `tensorchord/pgvecto-rs:pg16-v0.3.0` | `immich-postgres:5432` | — |
| redis | `redis:7-alpine` | `immich-redis:6379` | — |

- Storage: three `local-path` PVCs — postgres data (10Gi), upload/library (50Gi), ML model cache (10Gi)
- The server, machine-learning and redis Deployments use `Recreate` strategy; postgres runs as a StatefulSet
- Machine-learning models are downloaded on first start — expect several minutes before all pods become Ready

> **PostgreSQL storage must be local disk.** Network storage (NFS/CIFS) risks database corruption. Photo storage on the upload PVC can be on a network share if your `storageClassName` supports it.

## Quick start

```bash
# Install straight from the repo tarball (no clone needed)
helm install immich https://github.com/WOOWTECH/Woow_k3s_immich/archive/refs/heads/main.tar.gz

# Or from a local clone
git clone https://github.com/WOOWTECH/Woow_k3s_immich.git
cd Woow_k3s_immich
helm install immich .
```

> **Change the database password before any non-test deployment:**
>
> ```bash
> helm install immich . \
>   --set secrets.dbPassword="$(openssl rand -base64 24)"
> ```

Then open `http://<node-ip>:30283` and complete the Immich first-time setup.

## Key values

| Value | Default | Description |
|---|---|---|
| `namespace.create` / `namespace.name` | `true` / `immich` | Target namespace |
| `config.TZ` | `Asia/Taipei` | Container timezone (shared ConfigMap) |
| `server.image.tag` | `release` | Immich server version |
| `server.service.type` / `nodePort` | `NodePort` / `30283` | How Immich is exposed |
| `server.persistence.size` | `50Gi` | Upload/library PVC (`local-path`) |
| `machineLearning.enabled` | `true` | Deploy the ML worker |
| `machineLearning.persistence.size` | `10Gi` | Model cache PVC |
| `postgres.enabled` | `true` | Deploy the in-cluster PostgreSQL StatefulSet |
| `postgres.persistence.size` | `10Gi` | Database PVC (**local disk only**) |
| `redis.enabled` | `true` | Deploy the in-cluster Redis |
| `secrets.dbPassword` | `changeme-please` | PostgreSQL password |

Full list: [`values.yaml`](values.yaml)

## Verify

```bash
kubectl get pods -n immich          # four pods Running/Ready
curl http://<node-ip>:30283/api/server/ping
```

## Uninstall

```bash
helm uninstall immich
# PVCs are kept by Helm; remove them (and your data!) with:
kubectl delete pvc -n immich immich-postgres-data immich-upload-data immich-model-cache
```

## Migrating from the old Kustomize deployment

This repository replaces the `k3s` branch of the archived
[Woow_immich_docker_compose_all](https://github.com/WOOWTECH/Woow_immich_docker_compose_all)
repo. The chart's default rendering is resource-equivalent to those manifests
(same names, namespace, labels, ports, PVCs, kinds — postgres stays a
StatefulSet), so an existing deployment can be adopted by Helm or simply left
as-is; the original Kustomize files remain available in this repo's git
history.

The only intentional differences vs. the original manifests are:

- `imagePullPolicy` is written explicitly (`Always` for `:release`, `IfNotPresent` for pinned tags — this matches Kubernetes' implicit defaults)
- The namespace carries an extra `managed-by: helm` label

## License

MIT
