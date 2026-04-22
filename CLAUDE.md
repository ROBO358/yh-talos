# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

[talhelper](https://github.com/budimanjojo/talhelper) を使った Talos Linux クラスタの IaC 管理リポジトリ。
**Talos machine config とクラスタライフサイクル**、および **CNI (Cilium) のブートストラップ manifest** をここで管理する。
Kubernetes ワークロードおよび Cilium の日常運用（HelmRelease による管理）は [homelab-gitops](https://github.com/ROBO358/homelab-gitops) を参照。

- クラスタ名: `yh-cluster`
- 構成: Control Plane 3台 (c1-c3) + Worker 3台 (w1-w3)
- VIP: `192.168.1.200:6443`（Control Plane 3台が共有）
- Talos: `v1.12.6` / Kubernetes: `v1.35.2`
- CNI: Cilium（kube-proxy 置き換え）※ L2 Announcements は homelab-gitops の HelmRelease sync 後に有効化予定

## リポジトリ分担

| 責務 | yh-talos（ここ） | [homelab-gitops](https://github.com/ROBO358/homelab-gitops) |
|---|---|---|
| Talos machine config (`talconfig.yaml`) | ✓ | |
| クラスタ証明書・シークレット (SOPS) | ✓ | |
| クラスタライフサイクル (bootstrap/apply/upgrade/destroy/rebuild) | ✓ | |
| **CNI (Cilium) のブートストラップ manifest** | ✓ | |
| Flux v2 本体・Kustomization | | ✓ |
| **CNI (Cilium) の日常運用** (HelmRelease / CRD) | | ✓ |
| その他インフラ (Ingress, Storage, Secrets 等) | | ✓ |
| アプリケーションワークロード | | ✓ |

### CNI ブートストラップの方針

Cilium はクラスタの CNI として動作するため、**Flux が起動する前にクラスタネットワークが必要**という鶏と卵の問題がある。これを解決するため、Cilium の初期展開は Talos の `inlineManifests` 経由で行い、起動後は Flux の `HelmRelease` に管理を引き継ぐ。

**フロー**:

1. `task genconfig`: `cilium/values.yaml` を元に `helm template` で bootstrap manifest を生成し、`talconfig.yaml` の `cluster.inlineManifests` に埋め込む
2. `task bootstrap` / `task rebuild`: Talos がクラスタ初期化時に Cilium を展開 → クラスタネットワークが確立
3. Flux が homelab-gitops を sync → Cilium の `HelmRelease` が **既存リソースを引き継ぐ**
4. 以降、Cilium の設定変更・バージョンアップは homelab-gitops 側で管理

Talos の `inlineManifests` は「存在しないリソースのみ作成」する仕様のため、ステップ 3 で Flux の HelmRelease と管理権競合は発生しない。

### Cilium バージョンの同期

Cilium のバージョンは **両リポジトリで一致させる必要がある**:

- yh-talos: `Taskfile.yml` の `CILIUM_VERSION` 変数（`cilium/values.yaml` ではない）
- homelab-gitops: `infrastructure/cilium/controller/helmrelease.yaml` の `spec.chart.spec.version`

片方のみ更新すると次回リビルド時に想定外のバージョン変動が発生する。**両方を同じコミット/PR 単位で更新する**。

## よく使うコマンド

すべての操作は `task` (Taskfile) 経由で行う。

```bash
task                        # タスク一覧
task genconfig              # ノード設定ファイルを生成（変更なければスキップ）
task secrets:decrypt        # talsecret.sops.yaml の内容確認
task secrets:edit           # talsecret.sops.yaml をエディタで編集
task apply                  # 全ノードに設定を適用
task apply:node NODE=c1-k8s-yh  # 特定ノードのみ適用
task bootstrap              # クラスタ初期化（初回のみ・確認プロンプトあり）
task kubeconfig             # kubeconfig 取得
task upgrade:talos          # Talos アップグレード（全ノード）
task upgrade:talos:node NODE=c1-k8s-yh  # 特定ノードのみ Talos アップグレード
task upgrade:talos:node:preserve NODE=w1-k8s-yh  # Worker を --preserve 付きでアップグレード（Longhorn データ保護）
task upgrade:k8s            # Kubernetes アップグレード
task rebuild                # クラスタ再構築（destroy → bootstrap → kubeconfig）
task destroy                # 全ノードの EPHEMERAL 消去・再起動（IP・Machineconfig は保持）
task health                 # クラスタ全体のヘルスチェック（talosctl health）
task health:nodes           # 全ノードの Talos マシンステータス確認
task reboot:node NODE=192.168.1.201  # 特定ノードをリブート
task reboot:workers         # Worker を1台ずつ順番にリブート
task reboot:controlplanes   # CP を1台ずつリブート（etcd クォーラム維持）
```

## ファイル構成と役割

| ファイル | 役割 | コミット |
|---|---|---|
| `talconfig.yaml` | ノード定義・IP アドレス・バージョン・CNI/proxy 設定 | ✓ |
| `talsecret.sops.yaml` | クラスタシークレット（SOPS 暗号化済み） | ✓ |
| `.sops.yaml` | 暗号化ルール（公開鍵のみ・秘密情報なし） | ✓ |
| `Taskfile.yml` | タスク定義 | ✓ |
| `cilium/values.yaml` | Cilium ブートストラップ用 Helm values（securityContext・cgroup 等）。**チャートバージョンは含まない** | ✓ |
| `cilium/bootstrap.yaml` | `helm template` 出力（`task genconfig` が生成） | ✗（生成物） |
| `clusterconfig/*.yaml` | talhelper が生成するノード設定 | ✗（生成物） |
| `clusterconfig/talosconfig` | talhelper が生成するクライアント設定 | ✗（生成物） |

`clusterconfig/` および `cilium/bootstrap.yaml` は `task genconfig` で生成される。**直接編集しない**。

## シークレット管理

### アーキテクチャ

age 秘密鍵を 1Password に保管し、`op item get --reveal` で都度取得して `SOPS_AGE_KEY` 環境変数に渡す。

```
1Password (Private/talhelper-age-key)
    ↓ op item get --reveal
SOPS_AGE_KEY 環境変数（メモリのみ・使用後 unset）
    ↓
sops / talhelper genconfig
```

### ファイルの区別

- `talsecret.sops.yaml` … クラスタ証明書・トークン等の Credentials。SOPS 暗号化済みなのでコミット可。
- `.sops.yaml` … 暗号化ルールと age **公開鍵**のみ。Credentials ではない。
- `talsecret.yaml`（`.sops.` なし）… 平文の Credentials。作成・コミット禁止。

### 1Password CLI の注意点

実行環境は WSL2 のため `op` コマンドは Windows 版 `op.exe` のエイリアス。  
Taskfile の `vars.sh:` は非インタラクティブシェルで実行されるためエイリアスが展開されない。
`OP_CMD` 変数が起動時に `command -v op` で自動検出する（ネイティブ Linux `op` があれば `op`、なければ `op.exe`）。

`op run` は **使用不可**（Windows プロセスが WSL2 の Linux バイナリを起動できないため）。

## ノード情報

| ホスト名 | IP | 役割 |
|---|---|---|
| c1-k8s-yh | 192.168.1.201 | Control Plane（VIP ホルダー） |
| c2-k8s-yh | 192.168.1.202 | Control Plane |
| c3-k8s-yh | 192.168.1.203 | Control Plane |
| w1-k8s-yh | 192.168.1.211 | Worker |
| w2-k8s-yh | 192.168.1.212 | Worker |
| w3-k8s-yh | 192.168.1.213 | Worker |

## talconfig.yaml を編集するとき

- ノードの追加・変更後は必ず `task genconfig` → `task apply` の順で実行する
- バージョン変更（`talosVersion` / `kubernetesVersion`）後は `task upgrade:talos` / `task upgrade:k8s` を使う
- `task bootstrap` は **etcd を初期化する破壊的操作**。既存クラスタでは絶対に実行しない
- `cniConfig.name: none` および `cluster.proxy.disabled: true` は Cilium 前提の設定。変更しない
- **Cilium の日常の設定変更は homelab-gitops 側の HelmRelease で行う**。`cilium/values.yaml` はリビルド時のブートストラップ manifest 生成にのみ使われる
- Cilium のバージョンを更新する場合は `Taskfile.yml` の `CILIUM_VERSION` と homelab-gitops の `infrastructure/cilium/controller/helmrelease.yaml` を同じコミットで更新する（`cilium/values.yaml` はバージョンを含まない）

## Worker ノードと Longhorn

### system extensions と kubelet mount

Worker 3台（w1〜w3）は Longhorn の CSI ドライバに必要な拡張と bind mount が設定済み：

| 設定 | 値 | 理由 |
|---|---|---|
| `schematic.customization.systemExtensions` | `siderolabs/iscsi-tools`, `siderolabs/util-linux-tools` | iSCSI ボリューム操作 / fstrim |
| `kubelet.extraMounts` | `/var/mnt/longhorn` (bind/rshared/rw) | Longhorn データパス（Talos v1.10+ 推奨パス）|

CP ノードには上記設定は不要（Longhorn は Worker にのみ展開）。

### Worker の Talos アップグレード

**Longhorn インストール後は `task upgrade:talos` を使ってはいけない。** `upgrade:talos` は `--preserve` なしでアップグレードするため EPHEMERAL を wipe し、Longhorn のボリュームデータが消滅する。

Worker を個別にアップグレードする場合は必ず：

```bash
task upgrade:talos:node:preserve NODE=w1-k8s-yh
task upgrade:talos:node:preserve NODE=w2-k8s-yh
task upgrade:talos:node:preserve NODE=w3-k8s-yh
```

CP ノードのアップグレードは `task upgrade:talos:node NODE=c1-k8s-yh` で問題ない（CP は Longhorn データを持たない）。

### machine.files の注意点

`machine.files` の `op: create` は**ファイルを作成する**操作であり、ディレクトリは作成できない。`/var/mnt/longhorn` をディレクトリとして準備したい場合に `op: create` を使うと、同名のファイルが作成されて kubelet が起動失敗する。

`/var/mnt/longhorn` は containerd が kubelet コンテナ起動時に自動生成するため、`machine.files` による事前作成は不要。
