# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

[talhelper](https://github.com/budimanjojo/talhelper) を使った Talos Linux クラスタの IaC 管理リポジトリ。

- クラスタ名: `yh-cluster`
- 構成: Control Plane 3台 (c1-c3) + Worker 3台 (w1-w3)
- VIP: `192.168.1.200:6443`（Control Plane 3台が共有）
- Talos: `v1.12.6` / Kubernetes: `v1.35.2`

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
task upgrade:talos          # Talos アップグレード
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
| `talconfig.yaml` | ノード定義・IPアドレス・バージョン | ✓ |
| `talsecret.sops.yaml` | クラスタシークレット（SOPS暗号化済み） | ✓ |
| `.sops.yaml` | 暗号化ルール（公開鍵のみ・秘密情報なし） | ✓ |
| `Taskfile.yml` | タスク定義 | ✓ |
| `clusterconfig/*.yaml` | talhelper が生成するノード設定 | ✗（生成物） |
| `clusterconfig/talosconfig` | talhelper が生成するクライアント設定 | ✗（生成物） |

`clusterconfig/` 以下のファイルは `task genconfig` で生成される。**直接編集しない**。

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
