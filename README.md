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

By default (`secrets.create: false`) the chart never renders the database
Secret — it only references `immich-db-secret` by name, so an upgrade can
never overwrite a real password with an empty one. Create it once, out of
band, from [`examples/secrets.example.yaml`](examples/secrets.example.yaml):

```bash
kubectl create namespace immich
cp examples/secrets.example.yaml /secure/path/secrets.yaml   # outside the repo
# edit DB_PASSWORD in the copy, then:
kubectl apply -f /secure/path/secrets.yaml
```

Then install:

```bash
# Install straight from the repo tarball (no clone needed)
helm install immich https://github.com/WOOWTECH/Woow_k3s_immich/archive/refs/heads/main.tar.gz -n immich

# Or from a local clone
git clone https://github.com/WOOWTECH/Woow_k3s_immich.git
cd Woow_k3s_immich
helm install immich . -n immich
```

For a throwaway/test install, let the chart render the Secret instead:

```bash
helm install immich . -n immich \
  --set secrets.create=true \
  --set secrets.dbPassword="$(openssl rand -base64 24)"
```

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
| `keepOnUninstall` | `true` | Add `helm.sh/resource-policy: keep` to the Namespace, PVCs and Secret |
| `secrets.create` | `false` | Render the `immich-db-secret` Secret from `secrets.*` below instead of referencing an existing one |
| `secrets.dbPassword` | `""` | PostgreSQL password — **required** (fails the render) when `secrets.create=true`; never set a real value in this file |
| `tests.enabled` | `true` | Render the `helm test` smoke pod |

Full list: [`values.yaml`](values.yaml)

## Verify

```bash
kubectl get pods -n immich          # four pods Running/Ready
curl http://<node-ip>:30283/api/server/ping
helm test immich -n immich          # read-only smoke pod: ping + DB schema check
```

## Uninstall

By default (`keepOnUninstall: true`) the Namespace, the three PVCs, and the
Secret (when `secrets.create=true`) carry `helm.sh/resource-policy: keep`, so

```bash
helm uninstall immich -n immich
```

leaves your photos, the database, and the model cache in place — check with
`kubectl get pvc,secret,ns immich`. To actually delete everything:

```bash
kubectl delete pvc -n immich immich-postgres-data immich-upload-data immich-model-cache
kubectl delete secret -n immich immich-db-secret   # only if secrets.create=true was used
kubectl delete namespace immich
```

The keep policy is stamped onto the live objects at install/upgrade time, not
read at uninstall time — to opt out (e.g. disposable test installs) pass
`--set keepOnUninstall=false` to `helm install`/`helm upgrade` before you
uninstall.

## Migrating from the old Kustomize deployment

This repository replaces the `k3s` branch of the archived
[Woow_immich_docker_compose_all](https://github.com/WOOWTECH/Woow_immich_docker_compose_all)
repo. The chart's default rendering is resource-equivalent to those manifests
(same names, namespace, labels, ports, PVCs, kinds — postgres stays a
StatefulSet), so an existing deployment can be left as-is; the original
Kustomize files remain available in this repo's git history.

The intentional differences vs. the original manifests are:

- The Namespace, the three PVCs, and the Secret (when rendered) carry
  `helm.sh/resource-policy: keep` — see Uninstall above.
- The database password is no longer committed in `values.yaml`; by default
  the chart references an existing Secret instead of rendering one — see
  Quick start above.
- The Namespace carries an extra `managed-by: helm` label.
- `imagePullPolicy` is written explicitly: `Always` for the server and
  machine-learning Deployments (tag `:release`), `IfNotPresent` for postgres
  and redis (pinned tags). **This changes pull behaviour, it does not just
  spell out an implicit default:** Kubernetes' implicit `pullPolicy` is
  `IfNotPresent` for any tag other than `:latest`/no tag, `:release`
  included. The two `kubectl apply`-managed Immich deployments we found
  (neither has Helm ownership metadata, so `helm install` cannot adopt them
  as-is) were both observed running `IfNotPresent`. If you ever point this
  chart at a live deployment, override `server.image.pullPolicy` and
  `machineLearning.image.pullPolicy` to `IfNotPresent` first, or the next pod
  restart will jump straight to whatever `:release` currently points to.

## License

MIT
