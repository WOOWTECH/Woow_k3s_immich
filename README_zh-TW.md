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

預設(`secrets.create: false`)chart 不會渲染資料庫 Secret,只會用名稱
`immich-db-secret` 去參照既有的 Secret,這樣 `helm upgrade` 就永遠不會用空值
蓋掉真正的密碼。請先離線建立一次,來源是
[`examples/secrets.example.yaml`](examples/secrets.example.yaml):

```bash
kubectl create namespace immich
cp examples/secrets.example.yaml /secure/path/secrets.yaml   # 放在倉庫外
# 編輯這份複本裡的 DB_PASSWORD,然後:
kubectl apply -f /secure/path/secrets.yaml
```

接著安裝:

```bash
# 直接以倉庫 tarball 安裝(免 clone)
helm install immich https://github.com/WOOWTECH/Woow_k3s_immich/archive/refs/heads/main.tar.gz -n immich

# 或 clone 後安裝
git clone https://github.com/WOOWTECH/Woow_k3s_immich.git
cd Woow_k3s_immich
helm install immich . -n immich
```

若只是要用完即丟的測試安裝,可以改讓 chart 直接渲染 Secret:

```bash
helm install immich . -n immich \
  --set secrets.create=true \
  --set secrets.dbPassword="$(openssl rand -base64 24)"
```

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
| `keepOnUninstall` | `true` | 為 Namespace、PVC、Secret 加上 `helm.sh/resource-policy: keep` |
| `secrets.create` | `false` | 由 `secrets.*` 渲染 `immich-db-secret`,而非參照既有 Secret |
| `secrets.dbPassword` | `""` | PostgreSQL 密碼 —— `secrets.create=true` 時**必填**(否則 render 失敗);此檔案中永遠不要放真實密碼 |
| `tests.enabled` | `true` | 是否渲染 `helm test` smoke pod |

完整清單:[`values.yaml`](values.yaml)

## 驗證

```bash
kubectl get pods -n immich          # 四個 pod 均 Running/Ready
curl http://<node-ip>:30283/api/server/ping
helm test immich -n immich          # 唯讀 smoke pod:ping + 資料庫 schema 檢查
```

## 移除

預設(`keepOnUninstall: true`)Namespace、三個 PVC,以及 Secret(當
`secrets.create=true` 時)都帶有 `helm.sh/resource-policy: keep`,所以

```bash
helm uninstall immich -n immich
```

會保留相片、資料庫和模型快取 —— 用 `kubectl get pvc,secret,ns immich` 確認。
真的要全部刪除:

```bash
kubectl delete pvc -n immich immich-postgres-data immich-upload-data immich-model-cache
kubectl delete secret -n immich immich-db-secret   # 僅在曾用 secrets.create=true 時需要
kubectl delete namespace immich
```

keep policy 是在 install/upgrade 時寫進實際物件的,不是在 uninstall 當下讀取
——若要退出(例如用完即丟的測試安裝),請在 `helm install`/`helm upgrade`
時就加上 `--set keepOnUninstall=false`,不要等到要 uninstall 才設定。

## 從舊 Kustomize 部署遷移

本倉庫取代已封存的
[Woow_immich_docker_compose_all](https://github.com/WOOWTECH/Woow_immich_docker_compose_all)
`k3s` 分支。Chart 預設渲染結果與原 manifests 資源等價(名稱、namespace、標籤、
埠、PVC、kind 皆相同 —— postgres 維持 StatefulSet),既有部署可以維持原狀;
原始 Kustomize 檔案保留在本倉庫的 git 歷史中。

Chart 與原 manifests 的蓄意差異:

- Namespace、三個 PVC,以及 Secret(有渲染時)都帶
  `helm.sh/resource-policy: keep` —— 見上方「移除」。
- 資料庫密碼不再寫死在 `values.yaml`;預設改為參照既有 Secret,而非由 chart
  渲染 —— 見上方「快速開始」。
- Namespace 多帶 `managed-by: helm` 標籤。
- `imagePullPolicy` 明寫:server、machine-learning 兩個 Deployment(tag
  `:release`)是 `Always`;postgres、redis(固定 tag)是 `IfNotPresent`。
  **這會改變實際拉取行為,不只是把隱性預設寫出來:** K8s 對非 `:latest`/
  無 tag 的隱性預設一律是 `IfNotPresent`,`:release` 也不例外。我們找到的
  兩套 `kubectl apply` 部署(都沒有 Helm ownership metadata,`helm install`
  無法直接接管)實測都是 `IfNotPresent`。如果之後真的要用這個 chart 接管
  live 部署,請先把 `server.image.pullPolicy` 和
  `machineLearning.image.pullPolicy` 覆寫成 `IfNotPresent`,否則下一次 Pod
  重啟就會直接跳到 `:release` 當下指到的版本。

## 授權

MIT
