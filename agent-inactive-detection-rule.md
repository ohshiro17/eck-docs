# Elastic Agent 停止検知ルール 構築手順書

> 最終更新: 2026-03-26 | 作成者: ECK検証チーム

## 1. 概要

Elastic Agent がチェックインを停止した（inactive / offline 状態になった）ことを検知し、Webhook で外部に通知する仕組みの構築手順を記載する。

### 1.1 課題と要件

| 項目 | 内容 |
|------|------|
| 検知対象 | Elastic Agent が active 以外の状態（inactive, offline, unhealthy）になったこと |
| 通知方法 | Webhook（Slack, Teams 等に連携可能） |
| 制約 | 既存の inactive Agent が存在するため、単純な閾値（inactive 数 > 0）では誤検知する |
| 要件 | 「新たに停止した」エージェントのみを検知すること |

### 1.2 採用方式

**Kibana Alerting（Elasticsearch query ルール）** を採用した。

#### 方式比較（検討経緯）

| 方式 | ライセンス | 結果 |
|------|-----------|------|
| Elasticsearch Watcher | Platinum+ | search input がシステムインデックス `.fleet-agents` にアクセス不可。HTTP input は自己署名証明書の問題あり。**不採用** |
| Kibana Alerting（ES query） | Basic | Kibana 内部からシステムインデックスにアクセス可能。**採用** |
| CronJob + スクリプト | 不要 | Elastic 外の仕組みとなり管理が煩雑。**不採用** |

### 1.3 検知ロジック

Fleet は `.fleet-agents` インデックスでエージェント情報を管理しており、各エージェントは定期的にチェックインして `last_checkin` フィールドが更新される。

```
検知条件:
  active = true（登録上は有効なエージェント）
  AND last_checkin が 2分前〜10分前の範囲

→ 「直近までチェックインしていたが、2分以上停止した」エージェントを検出
```

**既存 inactive の除外**: `last_checkin` が10分以上前のエージェントはクエリ対象外となるため、既に長時間停止しているエージェントでは発火しない。

```
[時間軸]
|--- 10分前 ---|--- 2分前 ---|--- 現在 ---|
    ↑ 検知対象の範囲 ↑
                              ↑ 正常（まだチェックイン中）
↑ 対象外（既存の inactive）
```

## 2. 前提条件

- ECK on minikube 環境が構築済みであること
- Elasticsearch, Kibana, Fleet Server, Elastic Agent が稼働中であること
- Kibana にブラウザからアクセスできること

```bash
# Kibana へのアクセス
kubectl port-forward service/kibana-kb-http 5601:5601 -n elastic-stack
# ブラウザで https://localhost:5601 を開く
```

## 3. 構築手順

### 3.1 Connector の作成（通知先の設定）

#### 手順

1. Kibana にログインする
   - ユーザー: `elastic`
   - パスワード: Elasticsearch の Secret から取得
2. 左メニュー下部の **「Stack Management」** をクリック
3. 左サイドバーの **「Connectors」** をクリック
4. **「Create connector」** ボタンをクリック

#### 3.1.1 サーバーログ Connector（テスト用）

| 設定項目 | 入力値 |
|---------|--------|
| Connector type | **Server log** を選択 |
| Connector name | `Agent Inactive Log` |

→ **「Save」** をクリック

#### 3.1.2 Webhook Connector（本番通知用）

| 設定項目 | 入力値 |
|---------|--------|
| Connector type | **Webhook** を選択 |
| Connector name | `Agent Inactive Webhook` |
| Method | `POST` |
| URL | 通知先の Webhook URL（Slack, Teams 等） |
| Headers | Key: `Content-Type` / Value: `application/json` |

→ **「Save」** をクリック

### 3.2 Alerting Rule の作成（検知ルール）

#### 手順

1. 左メニュー下部の **「Stack Management」** をクリック
2. 左サイドバーの **「Rules」** をクリック
3. **「Create rule」** ボタンをクリック

#### 3.2.1 基本設定

| 設定項目 | 入力値 |
|---------|--------|
| Name | `Agent Inactive Detection` |
| Tags | `elastic-agent`, `monitoring` |
| Check every | `1 minute` |

#### 3.2.2 Rule type の選択

- **「Elasticsearch query」** を選択

#### 3.2.3 Query の設定

| 設定項目 | 入力値 |
|---------|--------|
| Select a data source | **「Elasticsearch query (DSL)」** を選択 |
| Index | `.fleet-agents` （手入力。システムインデックスのためサジェストに表示されない場合がある） |
| Time field | `last_checkin` |
| Size | `50` |

**Query** 欄に以下を入力:

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "active": true
          }
        },
        {
          "range": {
            "last_checkin": {
              "gte": "now-10m",
              "lte": "now-2m"
            }
          }
        }
      ]
    }
  }
}
```

#### 3.2.4 Condition の設定

| 設定項目 | 入力値 |
|---------|--------|
| When | **IS ABOVE** |
| Threshold | `0` |
| Time window | `10 minutes` |

→ **「Test query」** ボタンで動作確認が可能。正常時は `0 documents found` と表示される。

#### 3.2.5 Action の設定

**「Add action」** をクリック。

**サーバーログ（テスト用）:**

| 設定項目 | 入力値 |
|---------|--------|
| Action type | **Server log** |
| Connector | `Agent Inactive Log`（3.1.1 で作成したもの） |
| Run when | `Query matched` |
| Message | `[ALERT] Elastic Agent inactive detected. Count: {{context.value}}` |

**Webhook（本番用）:**

| 設定項目 | 入力値 |
|---------|--------|
| Action type | **Webhook** |
| Connector | `Agent Inactive Webhook`（3.1.2 で作成したもの） |
| Run when | `Query matched` |
| Body | 下記参照 |

Webhook Body の例:

```json
{
  "text": "[ALERT] Elastic Agent がチェックインを停止しました。検出数: {{context.value}} 件\n{{context.message}}"
}
```

#### 3.2.6 通知頻度の設定

| 設定項目 | 入力値 |
|---------|--------|
| Notify | **On active alerts** |
| Throttle | `10 minutes` |

→ **「Save」** をクリック

### 3.3 ルールの確認

保存後、Rules 一覧画面に `Agent Inactive Detection` が表示される。

| 確認項目 | 期待値 |
|---------|--------|
| Status | `Enabled` |
| Last response | `Ok`（初回実行後） |
| State | エージェントが正常な場合は空欄 |

## 4. 動作確認手順

### 4.1 Agent を意図的に停止

```bash
kubectl patch daemonset elastic-agent-agent -n elastic-stack \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-existing":"true"}}}}}'
```

### 4.2 アラート発火の確認（2〜3分後）

Kibana で **「Stack Management」→「Rules」** を開き、`Agent Inactive Detection` をクリック。

| 確認項目 | 期待値 |
|---------|--------|
| State | `Active` |
| Alerts | 停止したエージェントが表示される |
| Kibana サーバーログ | `[ALERT] Elastic Agent inactive detected. Count: 1` |

### 4.3 Agent の復旧

```bash
kubectl patch daemonset elastic-agent-agent -n elastic-stack \
  --type=json -p '[{"op":"remove","path":"/spec/template/spec/nodeSelector/non-existing"}]'
```

### 4.4 アラート解消の確認（2〜3分後）

| 確認項目 | 期待値 |
|---------|--------|
| State | 空欄（アクティブなアラートなし） |
| Alerts | `Recovered` と表示される |

## 5. 運用時の注意事項

### 5.1 時間窓パラメータの調整目安

| パラメータ | デフォルト | 調整の目安 |
|-----------|-----------|-----------|
| `lte: now-2m` | 2分 | Agent のチェックイン間隔（デフォルト60秒）の2倍。短すぎると一時的な遅延で誤検知 |
| `gte: now-10m` | 10分 | 長くすると既存 inactive を拾うリスクが上がる |
| Throttle | 10分 | 時間窓と合わせて同一エージェントの連続通知を防止 |

### 5.2 Webhook URL の変更方法

1. **「Stack Management」→「Connectors」** を開く
2. `Agent Inactive Webhook` をクリック
3. URL を変更して **「Save」**

### 5.3 ライセンスについて

Kibana Alerting の ES query ルール (`.es-query`) は **Basic ライセンスで利用可能**。
Watcher (Elasticsearch 側の alerting) は Platinum 以上が必要だが、本手順では使用していない。

> **Note**: 現在の検証環境は Trial ライセンス（有効期限: 2026-04-24）で動作確認済み。
> Basic ライセンスでの動作は、Trial 期限切れ後（自動的に Basic に戻る）に再確認を推奨する。
