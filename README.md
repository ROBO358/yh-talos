# yh-talos

Talos Linux cluster IaC for `yh-cluster`.

This repository manages:

- Talos machine configuration (via [talhelper](https://github.com/budimanjojo/talhelper))
- Cluster secrets (SOPS + age, age key in 1Password)
- Cluster lifecycle tasks (bootstrap, apply, upgrade, destroy, rebuild)
- **CNI (Cilium) bootstrap manifest** — rendered from `cilium/values.yaml` and embedded into Talos machine config via `inlineManifests`

Kubernetes workloads, Flux GitOps, and **day-2 Cilium management** (HelmRelease, CRDs) are in [homelab-gitops](https://github.com/ROBO358/homelab-gitops).

See [CLAUDE.md](./CLAUDE.md) for details on the repository split, Cilium bootstrap/handoff flow, and version synchronization rules.

## クラスタ再構築手順

### いつ使うか

| 状況 | 操作 |
|---|---|
| ノード設定を変更した（IP・CNI・バージョン等） | `task apply` |
| etcd・kubelet データをリセットして一から起動し直す | `task rebuild` |
| 設定変更 + リセット両方必要（上記組み合わせ） | `task apply` → `task rebuild` |
| シークレット（CA証明書・トークン）をローテーションしてリセット | `task secrets:rotate` → `task bootstrap` → `task kubeconfig` |

> **EPHEMERAL vs STATE**: `task rebuild` は EPHEMERAL パーティション（etcd・kubelet データ）のみ消去する。  
> STATE パーティション（Machineconfig・IP 設定）は保持されるため、事前に `task apply` で最新設定を書き込んでおく必要がある。

### 前提条件

- WSL2 から `192.168.1.0/24` への疎通確認（`talosctl version --nodes 192.168.1.201` で応答があること）
- 1Password CLI (`op.exe`) でログイン済み
- 必要ツール: `talosctl`, `talhelper`, `helm`, `task`, `yq`

### 手順

#### 1. 設定を全ノードの STATE に書き込む

`talconfig.yaml` や `cilium/values.yaml` を変更した場合、または最後の apply から時間が経っている場合に実行する。

```bash
task apply
```

- `talhelper genconfig` → `cilium:render`（helm template）→ 全 6 ノードへ apply の順で実行される
- "Applied configuration without a reboot" は設定変更なし（STATE と同じ内容）を意味する
- c1 で接続 timeout が出ても exit 0 なら問題ない（STATE は書き込まれている）

#### 2. クラスタを再構築する

```bash
task rebuild --yes
```

内部フロー（自動実行）:

1. **destroy**: 全 6 ノードの EPHEMERAL を並列に消去してリブート。各ノードを自身の IP でエンドポイント指定して独立して reset するため、1 ノードが再起動しても他ノードへの reset が影響を受けない
2. **ノード復帰待ち**: 全ノードが `talosctl version` に応答するまでポーリング
3. **bootstrap**: c1 で etcd を初期化。`cluster.inlineManifests` から Cilium を展開
4. **kubeconfig**: `~/.kube/config` に `admin@yh-cluster` コンテキストをマージ（既存は `admin@yh-cluster-1` にリネーム）

#### 3. コンテキストを切り替える

```bash
kubectl config use-context admin@yh-cluster
```

#### 4. 動作確認

```bash
# 全ノードが Ready（Cilium 起動まで数分かかる）
kubectl get nodes -o wide

# Cilium DaemonSet が全ノードで Running（6/6）
kubectl -n kube-system get ds cilium

# kube-proxy が存在しないこと（Cilium が置き換え）
kubectl -n kube-system get ds kube-proxy

# Cilium 内部ステータス
kubectl -n kube-system exec $(kubectl -n kube-system get pod -l k8s-app=cilium -o name | head -1) -- cilium status --brief

# CoreDNS が Running（Pod ネットワーク疎通の確認）
kubectl -n kube-system get pod -l k8s-app=kube-dns
```

### トラブルシューティング

**ノードが `talosctl version` に応答しない**  
ネットワーク疎通を確認する。WSL2 は再起動後にルーティングが消えることがある。

```bash
ip route show | grep 192.168
# 192.168.1.0/24 のルートがなければ Windows 側の VPN / ルーティング設定を確認
```

**bootstrap 後にノードが `NotReady` のまま**  
Cilium の起動に数分かかる。`kubectl -n kube-system get pod -l k8s-app=cilium` で Pod が `Running` になるまで待つ。

**一部ノードが `timed out` と表示された**  
destroy 後のウェイトループでタイムアウトしたノードは手動で復旧する。

```bash
# ノードの状態確認
talosctl --nodes 192.168.1.XXX --endpoints 192.168.1.XXX version

# 応答がなければ BMC/iDRAC でハードリセット
# 応答があれば手動でリブート
talosctl --nodes 192.168.1.XXX --endpoints 192.168.1.XXX reboot
```

全ノード復帰後、続きから再開できる:

```bash
task bootstrap --yes
task kubeconfig
```

---

## Roadmap

### Done
- [x] Talos cluster setup (3 control planes + 3 workers)
- [x] SOPS + age encryption for cluster secrets (age key stored in 1Password)
- [x] Taskfile automation (genconfig, apply, bootstrap, upgrade, health, reboot, destroy, rebuild)
- [x] Flux bootstrap → [homelab-gitops](https://github.com/ROBO358/homelab-gitops)

### In Progress

- [ ] **Cilium migration** — replace default flannel + kube-proxy with Cilium (CNI + kube-proxy replacement via KubePrism + L2 Announcements)
  - Bootstrap manifest generation is owned by this repo (`cilium/values.yaml` + `task genconfig`)
  - Day-2 operations and runtime configuration are in [homelab-gitops](https://github.com/ROBO358/homelab-gitops)

### Next (managed in homelab-gitops)

- [ ] Remove MetalLB once Cilium L2 Announcements is verified
- [ ] Ingress Controller (ingress-nginx) — HTTP/HTTPS routing
- [ ] Longhorn — persistent storage using node disks
- [ ] RBAC — cluster access control
- [ ] External Secrets Operator + 1Password — replace Kubernetes Secrets with 1Password-backed secrets (including Flux's own credentials)
