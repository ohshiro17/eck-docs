# ArgoCD を用いた ECK デプロイ自動化ガイド

## 目次

1. [現状構成の整理](#1-現状構成の整理)
1. [ArgoCD 導入後の構成](#2-argocd-導入後の構成)
1. [Application リソースのサンプル](#3-application-リソースのサンプル)
1. [移行手順](#4-移行手順)
1. [GitLab Helm リポジトリ認証トラブルシューティング](#5-gitlab-helm-リポジトリ認証トラブルシューティング)

-----

## 1. 現状構成の整理

### リポジトリ構成

|リポジトリ      |用途                          |認証方式        |
|-----------|----------------------------|------------|
|**GitLabA**|ECK の Helm チャート             |Deploy Token|
|**GitLabB**|values.yaml / pv.yaml / .env|アクセストークン    |

### 現状のデプロイフロー

```
GitLabA（Helmチャート）         GitLabB（設定ファイル）
  └ ECK公式Chart                  ├ values.yaml
    ※deploy tokenでアクセス        ├ pv.yaml（kind: PV）
                                  └ .env（バージョン等）
                                        ↓ 手動作業
                                  作業者のローカル環境
                                  $ source .env
                                  $ helm upgrade <release> <chart> \
                                      --version $ECK_VERSION \
                                      -f ./values.yaml
                                        ↓
                              クラスタB（ECK稼働）
```

### 課題

- デプロイが手動のため属人化している
- バージョン管理が `.env` ファイルのみで、デプロイ履歴が追いにくい
- GitLabA の Helm リポジトリへの認証が不安定

-----

## 2. ArgoCD 導入後の構成

### クラスタ構成

|クラスタ     |役割        |管理者   |
|---------|----------|------|
|**クラスタA**|ArgoCD が稼働|別チーム管理|
|**クラスタB**|ECK が稼働   |自チーム管理|


> **制約**: クラスタA への直接デプロイは不可。ArgoCD の Application リソースの登録のみ許可。

### 導入後のリポジトリ構成（GitLabB）

```
GitLabB/
  ├ argocd/
  │   └ eck-application.yaml   # Applicationリソース（手動 apply）
  ├ eck/
  │   └ values.yaml            # ECK専用のvalues
  ├ pv.yaml                    # PVマニフェスト
  └ .env                       # バージョン管理（記録用）
```

### 導入後のデプロイフロー

```
.envのECK_VERSIONを更新
    ↓
eck-application.yamlのtargetRevisionを更新してGitLabBにPush
    ↓
ArgoCDがGitLabBの変更を検知
    ↓
ArgoCDがGitLabAから該当バージョンのChartを取得
    ↓
クラスタBに自動適用 ✅
```

### クラスタB の登録

ArgoCD にクラスタB を登録する際は以下コマンドを実行する（初回のみ）。

```bash
argocd cluster add <context-name>
```

このコマンドの動作：

1. 指定した kubeconfig で一時的にクラスタB へ接続
1. クラスタB に `argocd-manager` ServiceAccount と ClusterRoleBinding を作成
1. その ServiceAccount のトークンを ArgoCD の Secret に保存

> **注意**: 初回登録後は kubeconfig の有効期限は関係なくなる。以降は ArgoCD が保持する ServiceAccount トークンで通信する。

-----

## 3. Application リソースのサンプル

### multiSource を使った構成

GitLabA の Helm チャートと GitLabB の values.yaml を同時に参照する。

```yaml
# argocd/eck-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: eck
  namespace: argocd
spec:
  project: default
  sources:
    # GitLabA: Helmチャート
    - repoURL: https://gitlab.example.com/api/v4/projects/<PROJECT_ID>/packages/helm/stable
      chart: eck-stack
      targetRevision: "2.9.0"       # .envのECK_VERSIONに合わせて更新
      helm:
        valueFiles:
          - $values/eck/values.yaml  # GitLabBのvaluesを参照
    # GitLabB: values参照用
    - repoURL: https://gitlab.example.com/your-team/gitlabB.git
      targetRevision: main
      ref: values                    # 上の$valuesの参照名

  destination:
    server: https://<クラスタBのAPIサーバーURL>
    namespace: elastic

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### PV の Application リソース

PV は Helm チャートとは別の Application で管理する。

```yaml
# argocd/pv-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: eck-pv
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.example.com/your-team/gitlabB.git
    targetRevision: main
    path: .                          # pv.yamlが置いてある場所
  destination:
    server: https://<クラスタBのAPIサーバーURL>
    namespace: elastic
  syncPolicy:
    automated:
      prune: false                   # PVは誤削除防止のためpruneをオフ
      selfHeal: true
```

### apply 方法

```bash
kubectl apply -f argocd/eck-application.yaml -n argocd
kubectl apply -f argocd/pv-application.yaml -n argocd
```

-----

## 4. 移行手順

### Step 1: GitLabB のリポジトリ構成を整理

```bash
mkdir -p argocd eck

# valuesをeck/配下に移動
mv values.yaml eck/values.yaml

# Applicationリソースファイルを作成
touch argocd/eck-application.yaml
touch argocd/pv-application.yaml
```

### Step 2: GitLabA の Helm リポジトリを ArgoCD に登録

別チームの ArgoCD 管理者に依頼、または Settings > Repositories から登録。

```
Repository Type : Helm
Repository URL  : https://gitlab.example.com/api/v4/projects/<PROJECT_ID>/packages/helm/stable
Username        : gitlab-deploy-token
Password        : <Deploy Tokenの値>
```

> 認証エラーが発生する場合は [セクション5](#5-gitlab-helm-リポジトリ認証トラブルシューティング) を参照。

### Step 3: GitLabB を ArgoCD に登録

```
Repository Type : Git
Repository URL  : https://gitlab.example.com/your-team/gitlabB.git
Username        : <GitLabユーザー名>
Password        : <アクセストークン>
```

### Step 4: クラスタB を ArgoCD に登録

```bash
# kubeconfigのコンテキスト名を確認
kubectl config get-contexts

# クラスタBを登録
argocd cluster add <クラスタBのコンテキスト名>

# 登録確認
argocd cluster list
```

### Step 5: Application リソースを apply

```bash
# GitLabBをclone
git clone https://gitlab.example.com/your-team/gitlabB.git
cd gitlabB

# Applicationリソースを記述して apply
kubectl apply -f argocd/eck-application.yaml -n argocd
kubectl apply -f argocd/pv-application.yaml -n argocd
```

### Step 6: ArgoCD の同期確認

```bash
# Applicationの状態確認
argocd app list

# 詳細確認
argocd app get eck
```

`STATUS` が `Synced`、`HEALTH` が `Healthy` になれば成功。

### Step 7: 手動デプロイの廃止

ArgoCD による自動同期が確認できたら、手動の `helm upgrade` コマンドでの運用を終了する。

### バージョン更新時の手順（移行後の日常運用）

```bash
# 1. .envとeck-application.yamlのtargetRevisionを更新
vi .env                              # ECK_VERSION=2.10.0
vi argocd/eck-application.yaml       # targetRevision: "2.10.0"

# 2. GitLabBにPush
git add .
git commit -m "chore: bump ECK version to 2.10.0"
git push origin main

# 3. ArgoCDが自動検知して適用（手動トリガーも可能）
argocd app sync eck
```

-----

## 5. GitLab Helm リポジトリ認証トラブルシューティング

### 前提確認

ArgoCD の Settings > Repositories で Failed になる場合、以下を順番に確認する。

### チェック1: Helm リポジトリの URL 形式

GitLab の Helm リポジトリ URL はプロジェクト ID が必要。

```bash
# ❌ よくある間違い
https://gitlab.example.com/group/project

# ✅ 正しい形式
https://gitlab.example.com/api/v4/projects/<PROJECT_ID>/packages/helm/stable
```

プロジェクト ID は GitLab のプロジェクトページ上部（プロジェクト名の下）に表示されている数字。

### チェック2: Deploy Token のスコープ

GitLab の Deploy Token に **`read_package_registry`** スコープが付与されているか確認する。

`read_repository` のみでは Helm パッケージレジストリにアクセスできない。

```
GitLab > Project > Settings > Repository > Deploy tokens
  └ Scopes
      ☑ read_package_registry   ← これが必要
      □ read_repository
```

### チェック3: Username の設定

|Username に入力する値      |説明                  |
|---------------------|--------------------|
|`gitlab-deploy-token`|Deploy Token 使用時の固定値|
|GitLab ユーザー名         |アクセストークン使用時         |

Deploy Token 使用時は Username を `gitlab-deploy-token`（固定値）にする。

### チェック4: SSL 証明書の検証エラー

GitLab がプライベート証明書や自己署名証明書を使っている場合、ArgoCD が SSL 検証で弾く。

UI登録時に **「Skip server verification」** をチェックして再試行する。

恒久対応として ArgoCD に CA 証明書を登録する場合：

```bash
argocd cert add-tls gitlab.example.com --from /path/to/ca.crt
```

### チェック5: ログの確認

より詳細なエラーは `argocd-repo-server` のログに出る（`argocd-server` ではない）。

```bash
kubectl logs -n argocd deploy/argocd-repo-server | grep -i "error\|failed"
```
## 6. ArgoCD と ECK のラベル競合対策

### 問題

ArgoCD はリソース管理のために `app.kubernetes.io/instance` ラベルを自動付与する。
ECK Operator も同じラベルを使用するため、Sync のたびに競合が発生する。

### 解決策

`ignoreDifferences` と `RespectIgnoreDifferences=true` をセットで設定する。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: eck
  namespace: argocd
spec:
  ignoreDifferences:
    - group: elasticsearch.k8s.elastic.co
      kind: Elasticsearch
      jsonPointers:
        - /metadata/labels/app.kubernetes.io~1instance
  syncPolicy:
    syncOptions:
      - RespectIgnoreDifferences=true
```

# Helmリリース名の指定について

ArgoCDでHelmチャートをデプロイする際、デフォルトではArgoCDのアプリケーション名がそのままHelmのリリース名になる。

既存のHelmリリースを引き継ぐ場合は、`spec.source.helm.releaseName`を明示的に指定する。

```yaml
spec:
  source:
    helm:
      releaseName: eck-stack  # 既存のHelmリリース名を指定
```

|項目                         |値          |
|---------------------------|-----------|
|ArgoCDアプリ名（`metadata.name`）|`dev-stack`|
|Helmリリース名（`releaseName`）   |`eck-stack`|

`releaseName`を省略すると、`dev-stack`という名前で新規リリースが作られ、既存の`eck-stack`とは別に二重デプロイされてしまう。

## `syncPolicy` への `ApplyOutOfSyncOnly` 追加

手動Sync時にOutOfSyncのリソースのみを適用するよう、`eck-application.yaml` の `syncPolicy` に以下を追記する。

```yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ApplyOutOfSyncOnly=true  # OutOfSyncのリソースのみSyncの対象にする
```

> **補足**: デフォルトではSync時にアプリケーション内の全リソースが `kubectl apply` される。
> `ApplyOutOfSyncOnly=true` を設定することで、差分があるリソースのみが適用され、既存リソースへの意図しない上書きを防ぐことができる。


### エラー別対処表

|エラーメッセージ            |原因              |対処                         |
|--------------------|----------------|---------------------------|
|`401 Unauthorized`  |認証情報が間違っている     |Username/Password を再確認     |
|`403 Forbidden`     |スコープ不足          |`read_package_registry` を付与|
|`404 Not Found`     |URL が間違っている     |PROJECT_ID を含む URL に修正     |
|`x509: certificate` |SSL 証明書エラー      |Skip verification または CA 登録|
|`connection refused`|URL のホスト名が間違っている|URL を再確認                   |
