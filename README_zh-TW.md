# Woow_k3s_immich — Immich 的 K3s/Kubernetes Helm Chart

[English](README.md)

在 K3s/Kubernetes 上部署 [Immich](https://immich.app) 的 Helm chart —— 自架式
照片與影片管理平台。內含 Immich server、machine-learning 工作者(臉部辨識、
智慧搜尋、自動標籤)、in-cluster Redis,以及 `pgvecto.rs` PostgreSQL StatefulSet。

> **要用其他平台部署?**
> Docker/Podman Compose → [Woow_podman_immich](https://github.com/WOOWTECH/Woow_podman_immich) ·
> Home Assistant add-on → [Woow_ha_immich](https://github.com/WOOWTECH/Woow_ha_immich)

## 架構

| 元件 | 映像 | Service | NodePort |
|---|---|---|---|
| immich-server | `ghcr.io/immich-app/immich-server:release` | `immich-server:2283` | `30283` |
| immich-machine-learning | `ghcr.io/immich-app/immich-machine-learning:release` | `immich-machine-learning:3003` | — |
| postgres(pgvecto.rs) | `tensorchord/pgvecto-rs:pg16-v0.3.0` | `immich-postgres:5432` | — |
| redis | `redis:7-alpine` | `immich-redis:6379` | — |

- 儲存:三個 `local-path` PVC — postgres 資料(10Gi)、上傳/相片庫(50Gi)、ML 模型快取(10Gi)
- server、machine-learning、redis 三個 Deployment 用 `Recreate` 策略;postgres 為 StatefulSet
- 首次啟動時 machine-learning 會下載模型,所有 pod 就緒需等待數分鐘

> **PostgreSQL 資料必須放本機磁碟。** NFS/CIFS 等網路儲存會導致資料庫損毀。上傳 PVC 若使用支援網路儲存的 `storageClassName` 則相片可以放遠端。

## 快速開始

```bash
# 直接以倉庫 tarball 安裝(免 clone)
helm install immich https://github.com/WOOWTECH/Woow_k3s_immich/archive/refs/heads/main.tar.gz

# 或 clone 後安裝
git clone https://github.com/WOOWTECH/Woow_k3s_immich.git
cd Woow_k3s_immich
helm install immich .
```

> **非測試環境部署前務必更換資料庫密碼:**
>
> ```bash
> helm install immich . \
>   --set secrets.dbPassword="$(openssl rand -base64 24)"
> ```

完成後開啟 `http://<node-ip>:30283`,並完成 Immich 首次設定精靈。

## 主要設定值

| 設定 | 預設 | 說明 |
|---|---|---|
| `namespace.create` / `namespace.name` | `true` / `immich` | 目標 namespace |
| `config.TZ` | `Asia/Taipei` | 容器時區(共用 ConfigMap) |
| `server.image.tag` | `release` | Immich server 版本 |
| `server.service.type` / `nodePort` | `NodePort` / `30283` | Immich 對外方式 |
| `server.persistence.size` | `50Gi` | 上傳/相片庫 PVC(`local-path`) |
| `machineLearning.enabled` | `true` | 是否部署 ML 工作者 |
| `machineLearning.persistence.size` | `10Gi` | 模型快取 PVC |
| `postgres.enabled` | `true` | 是否部署 in-cluster PostgreSQL StatefulSet |
| `postgres.persistence.size` | `10Gi` | 資料庫 PVC(**限本機磁碟**) |
| `redis.enabled` | `true` | 是否部署 in-cluster Redis |
| `secrets.dbPassword` | `changeme-please` | PostgreSQL 密碼 |

完整清單:[`values.yaml`](values.yaml)

## 驗證

```bash
kubectl get pods -n immich          # 四個 pod 均 Running/Ready
curl http://<node-ip>:30283/api/server/ping
```

## 移除

```bash
helm uninstall immich
# Helm 會保留 PVC;確定不要資料後再刪:
kubectl delete pvc -n immich immich-postgres-data immich-upload-data immich-model-cache
```

## 從舊 Kustomize 部署遷移

本倉庫取代已封存的
[Woow_immich_docker_compose_all](https://github.com/WOOWTECH/Woow_immich_docker_compose_all)
`k3s` 分支。Chart 預設渲染結果與原 manifests 資源等價(名稱、namespace、標籤、
埠、PVC、kind 皆相同 —— postgres 維持 StatefulSet),既有部署可交由 Helm 接管
或維持原狀;原始 Kustomize 檔案保留在本倉庫的 git 歷史中。

Chart 與原 manifests 唯二的蓄意差異:

- `imagePullPolicy` 明寫(`:release` → `Always`、固定 tag → `IfNotPresent`,與 K8s 隱性預設一致)
- Namespace 多帶 `managed-by: helm` 標籤

## 授權

MIT
