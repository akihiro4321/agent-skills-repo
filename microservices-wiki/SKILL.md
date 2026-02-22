---
name: microservices-wiki
description: マイクロサービスアーキテクチャ全体の俯瞰的なWikiドキュメントを自動生成するスキル。インフラ定義ファイル（docker-compose, k8s, Terraform）・API仕様（OpenAPI, proto）・DB構成・CI/CDを横断的に分析し、サービス間の連携・データフロー・インフラ構成を包括的にドキュメント化する。ユーザーが「マイクロサービスのアーキテクチャをドキュメント化して」「サービス全体の構成を把握したい」と依頼した場合に使用する。
---

# microservices-wiki - マイクロサービス全体アーキテクチャ Wiki 生成スキル

インフラ定義・API仕様・DB構成を分析し、マイクロサービス全体の俯瞰的なWikiドキュメントを生成する。**個別サービスの内部実装ではなく、サービス間の連携・データフロー・インフラ構成に特化する。**

> **品質目標**: このWikiを読むだけで、新規参画者がシステム全体の構成・サービス間の通信経路・データの流れ・デプロイ構成を把握できること。また、障害発生時の影響範囲特定・新規サービス追加時の設計判断に利用できること。

> **スコープの明確化**:
> - ✅ このスキルが扱うもの: サービス間の依存関係・通信プロトコル・データストア構成・インフラ・横断的関心事
> - ❌ このスキルが扱わないもの: 各サービスの内部実装・ビジネスロジックの詳細（それは deepwiki スキルの領域）

> **言語ルール**: ページタイトルは**英語**。本文は**日本語**。コード要素・ファイルパス・コマンドは英語のまま。

参照ドキュメント: テンプレートは [references/prompts.md](references/prompts.md)、出力フォーマットは [references/output-format.md](references/output-format.md)

---

## Phase 1: インプット収集

**コンテキスト制御の原則**: ソースコード全体を読まない。インフラ定義・API仕様・DB定義・既存ドキュメントに限定することで、コンテキストウィンドウ内に収める。

### Step 1a: インフラ定義ファイルの収集

まずスクリプトを実行して対象の全体像を把握する：

```bash
bash scripts/collect_infra.sh <ルートパス>
```

**スクリプトが収集する情報**:
- サービス一覧（docker-compose の `services:` / k8s の `Deployment` リソース）
- 各サービスのポートマッピング・環境変数（他サービスのURL等）
- ネットワーク構成（docker network / k8s namespace）
- インフラ関連ファイルの一覧（terraform, helm chart 等）

**スクリプト実行後、以下を手動で読む（存在するものだけ）**:

| 優先度 | ファイル種別 | 対象パターン |
| :--- | :--- | :--- |
| 最高 | コンテナ構成 | `docker-compose*.yml`, `docker-compose*.yaml` |
| 最高 | k8s構成 | `k8s/**/*.yaml`, `kubernetes/**/*.yaml`, `helm/**/values.yaml` |
| 高 | API Gateway | `api-gateway/**/*`, `nginx/**/*.conf`, `traefik/**/*`, `kong/**/*` |
| 高 | インフラ定義 | `terraform/**/*.tf`（`main.tf`, `variables.tf` 優先） |
| 中 | CI/CD | `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile` |
| 中 | サービスメッシュ | `istio/**/*`, `linkerd/**/*` |

> **注意**: ファイルが多い場合は全て読む必要はない。サービス間の依存関係把握に必要な情報を優先する。

### Step 1b: API仕様の収集

サービス間のI/Fを把握する最重要インプット。

1. OpenAPI/Swagger 仕様を検索して読む：
   ```
   openapi*.yaml, openapi*.json, swagger*.yaml, swagger*.json
   api-docs/*.yaml, docs/api/*.yaml
   ```

2. gRPC を利用している場合は proto ファイルを読む：
   ```
   **/*.proto, proto/**/*.proto
   ```

3. **API仕様が存在しない場合**: 各サービスのルーターファイルを限定的に読む
   - Express: `routes/*.ts`, `src/routes/*.ts`
   - FastAPI/Flask: `routers/*.py`, `api/*.py`
   - Spring: `*Controller.java`
   - 目的はエンドポイントの一覧把握のみ。実装詳細は読まない。

### Step 1c: DB・データストア構成の把握

| 対象 | 優先して読むファイル |
| :--- | :--- |
| RDB スキーマ | `migrations/*.sql`, `schema.sql`, `**/schema.prisma`, `db/migrate/*.rb` |
| NoSQL 構成 | `docker-compose.yml` の MongoDB/DynamoDB 設定 |
| キャッシュ | Redis/Memcached の設定ファイル |
| メッセージキュー | Kafka topic 定義, `**/kafka/**/*.yaml`, RabbitMQ の exchange 定義 |

> **目的**: どのサービスがどのDBを使うか、DBが共有されているかを把握する。スキーマ全体の詳細は不要。

### Step 1d: 既存ドキュメントの収集

以下が存在する場合は読む（コンテキストの補完に使う）：

```
README.md（ルート）, ARCHITECTURE.md, docs/architecture/*, ADR（Architecture Decision Records）
各サービスの docs/wiki/index.md（deepwikiで生成済みの場合）
```

### Step 1e: アーキテクチャ概要メモの生成

Step 1a〜1d の結果を以下の形式で整理する。**これが Phase 2・3 の入力になる。**

```text
## アーキテクチャ概要メモ

### 検出されたサービス一覧
| サービス名 | 技術スタック | ポート | DB | 役割（推定） |
| :--- | :--- | :--- | :--- | :--- |
| user-service | Node.js | 3001 | PostgreSQL | ユーザー認証・管理 |
| order-service | Python | 8000 | MySQL | 注文管理 |
| notification-service | Go | 8080 | Redis | 通知送信（非同期） |

### サービス間通信
- [同期/REST]: [呼び出し元] → [呼び出し先]: [エンドポイント] (環境変数: USER_SERVICE_URL等から判断)
  例: order-service → user-service: GET /users/{id} (USER_SERVICE_URL から判断)
- [同期/gRPC]: [呼び出し元] → [呼び出し先]: [ServiceName.Method]
- [非同期/MQ]: [Publisher] → [topic/queue名] → [Subscriber]

### データストア構成
- [サービス名]: 専有 [DB種別] (schema: [主要テーブル])
- [サービス名]: 専有 [DB種別] + 共有 [キャッシュ] with [サービス名]
- 共有DBがある場合: ⚠️ [サービスA] と [サービスB] が同一[DB名]を共有

### API Gateway / エントリーポイント
- ゲートウェイ: [技術名] (パス: [設定ファイルパス])
- ルーティング:
  - /api/users/* → user-service:3001
  - /api/orders/* → order-service:8000

### 横断的関心事
- 認証・認可: [方式] (例: JWT, OAuth2, API Key)
- サービスディスカバリ: [方式] (例: k8s DNS, Consul)
- ログ集約: [ツール] (例: ELK Stack, Datadog)
- トレーシング: [ツール] (例: Jaeger, Zipkin, OpenTelemetry)
- CI/CD: [ツール] (例: GitHub Actions, Jenkins)

### 注目すべき問題・特徴
- ⚠️ [共有DB等の問題点]: [詳細]
- ✅ [良い設計]: [詳細]
```

---

## Phase 2: アーキテクチャ分析

Phase 1 のメモを基に、以下の分析を深める。

### サービス依存関係グラフの精緻化

1. **環境変数からの依存推定**: `*_URL`, `*_HOST`, `*_ENDPOINT` 等の環境変数を確認し、サービス間のURL参照を特定する
2. **API Gatewayルーティングの確認**: ゲートウェイ設定から外部→内部のルーティングを整理
3. **非同期依存の特定**: Kafka/RabbitMQ のtopic/exchange名を検索し、Publisher/Subscriber の関係を明確化
4. **DBの共有・専有判定**: 同一DB接続文字列・ホスト名が複数サービスで共有されていないか確認

### アーキテクチャパターンの検出

| パターン | 検出シグナル |
| :--- | :--- |
| API Gateway | nginx/kong/traefik の設定, `api-gateway` ディレクトリ |
| BFF (Backend for Frontend) | `web-bff`, `mobile-bff` 等のサービス名 |
| Saga パターン | `saga`, `choreography`, `orchestration` キーワード |
| CQRS | `command`, `query` の分離、読み書きDBの分離 |
| Event Sourcing | `event-store`, `EventStore` キーワード |
| Circuit Breaker | `hystrix`, `resilience4j`, `retry` 設定 |
| Service Mesh | Istio/Linkerd 設定ファイルの存在 |

---

## Phase 3: Wiki構造設計 + ユーザー確認

Phase 1-2 の分析を基に、全体アーキテクチャ Wiki の構造を設計する。

### セクション構成テンプレート（5セクション基本形）

対象システムの実態に合わせて調整する。存在しない関心事のセクションは省略してよい。

```
1. System Overview（システム概要）
   1.1 Architecture Overview（全体アーキテクチャ概要）  ← 必須・importance: high
   1.2 Service Catalog（サービス一覧・責務定義）       ← 必須・importance: high

2. Service Communication（サービス間通信）
   2.1 API Gateway & Routing（APIゲートウェイ・ルーティング）
   2.2 Synchronous Communication（同期通信: REST/gRPC）
   2.3 Asynchronous Communication（非同期通信: MQ/Event Streaming）
   ※ 非同期通信がない場合は 2.3 を省略

3. Data Architecture（データアーキテクチャ）
   3.1 Database per Service（サービス別DB設計）
   3.2 Data Flow & Consistency（データフロー・整合性）
   ※ 共有DBがない場合は 3.2 を簡略化

4. Infrastructure & Deployment（インフラ・デプロイ）
   4.1 Container Orchestration（コンテナ構成）
   4.2 Service Mesh & Networking（ネットワーク構成）   ← サービスメッシュがある場合のみ
   4.3 CI/CD Pipeline（デプロイパイプライン）

5. Cross-Cutting Concerns（横断的関心事）
   5.1 Authentication & Authorization（認証・認可）
   5.2 Observability（可観測性: ログ・メトリクス・トレーシング）
   5.3 Error Handling & Resilience（エラー処理・耐障害性）   ← Circuit Breaker等がある場合
```

### ページ定義の形式

```json
{
  "id": "1.1",
  "title": "Architecture Overview",
  "inputSources": [
    "docker-compose.yml",
    "k8s/deployments/*.yaml",
    "docs/architecture/README.md"
  ],
  "importance": "high",
  "relatedPages": ["1.2", "2.1"],
  "keyDiagrams": ["全体構成 flowchart", "サービス間通信 sequenceDiagram"],
  "linkedServiceWikis": ["user-service/docs/wiki/1.1-architecture-overview.md"]
}
```

> [!CAUTION]
> **【超重要: ユーザー確認の必須化】**
> 構造を JSON 形式でアウトラインとして生成した後、**絶対に Phase 4 (ページ作成) に進んではいけません。**
> 必ずユーザーに対して「この構造でページ生成を開始してよいか」の承認を求めてください。

### ページ数ガイドライン

| サービス数 | 推奨ページ数 | 備考 |
| :--- | :--- | :--- |
| 3-5サービス | 10-18ページ | 小規模 |
| 6-15サービス | 18-30ページ | 中規模 |
| 16サービス以上 | 30-50ページ | 大規模: サービスグループ単位でページ追加 |

---

## Phase 4: ページ生成ループ

> **核心ルール: 全ページを一括生成しない。1ページずつ「分析→生成→出力→検証」のループを回す。**

importance の順に処理: `high` → `medium` → `low`

### 各ページの処理手順

#### Step 4a: ソース再確認（ページ固有の分析）

各ページの `inputSources` に記載されたファイルを再確認する。

1. Phase 1 で読んだ情報で足りなければ、該当設定ファイルを再度読む
2. **スニペット候補を 5-10 個リストアップ** してから生成に進む
   - インフラ定義のスニペット例: docker-compose のサービス定義、k8s の Deployment/Service 定義、nginx のルーティング設定等
   - API仕様のスニペット例: OpenAPI のエンドポイント定義、proto のサービス定義

#### Step 4b: ページ生成

分析メモを基に Markdown ページを生成する。

**アーキテクチャWiki特有の品質基準**:

1. **Mermaid ダイアグラム（必須）**: importance: high は **2-3個、2種類以上**
   - 全体構成には `flowchart TD` または `graph TD`（**`LR` は絶対禁止**）
   - サービス間通信には `sequenceDiagram`
   - DB構成には `erDiagram`
   - 状態・ライフサイクルには `stateDiagram-v2`
   - **ノードラベルには実際のサービス名・コンポーネント名**を使う

   > [!CAUTION]
   > **【Mermaid パースエラー防止: 絶対厳守】**（詳細は [references/prompts.md](references/prompts.md) を参照）
   > - ノードラベルに `()`, `[]`, `{}`, `<>` を含む場合は **必ず `["ラベル"]` でダブルクォートで囲む**
   >   - ❌ `D[Data Pipeline (api.py)]` → ✅ `D["Data Pipeline (api.py)"]`
   > - **ノードIDにハイフンを使用禁止**（アンダースコアを使用）
   >   - ❌ `order-service["order-service"]` → ✅ `order_svc["order-service"]`
   > - HTMLタグ（`<`, `>` 等）はノードラベルに使用不可
   > - `sequenceDiagram` の矢印はフローチャート記法 `--|label|-->` を**絶対に使用しない**。必ず `A->>B: label` 形式で書く

2. **コードスニペット（重要）**: 設定ファイルからの実際の引用のみ（疑似コード禁止）
   - `docker-compose.yml` のサービス定義
   - k8s マニフェスト（Deployment, Service, Ingress）
   - nginx/kong ルーティング設定
   - OpenAPI エンドポイント定義
   - proto サービス定義

3. **Sources 行**: 設定ファイルへの参照（行番号付き）
   - 形式: `**Sources:** [docker-compose.yml:L1-L45](file:///絶対パス/docker-compose.yml#L1-L45)`
   - 既存の個別 Wiki がある場合: `[user-service Architecture](file:///path/docs/wiki/1.1-architecture-overview.md)`

4. **テーブル**: サービス一覧・API一覧・DB一覧等はテーブルで整理

#### Step 4c: ファイル出力

生成したページを即座にファイルに書き出す。

#### Step 4d: バリデーション

```bash
python3 scripts/validate_arch_page.py <生成したファイル.md> --importance <high|medium|low>
```

- **Grade B 以上 (75%+)**: 合格。次のページへ進む。
- **Grade C (60-74%)**: 指摘項目を修正して再検証。最大2回まで修正ループ。
- **Grade D 以下 (<60%)**: Step 4a に戻り、ソース再確認・再生成。

> [!CAUTION]
> バリデーションを後回しにして複数ページを一気に生成することを**絶対に行ってはいけません**。

**全ページ完了後のディレクトリ全体検証**:

```bash
python3 scripts/validate_arch_page.py <arch-wiki出力ディレクトリ> --scale <small|medium|large>
```

---

## Phase 5: 出力

デフォルトの出力先: `<ルートパス>/docs/arch-wiki/`

```
docs/arch-wiki/
├── index.md                          # メインインデックス
├── 1-system-overview.md
├── 1.1-architecture-overview.md      # ← 最重要ページ
├── 1.2-service-catalog.md
├── 2-service-communication.md
├── 2.1-api-gateway-and-routing.md
├── 2.2-synchronous-communication.md
├── 2.3-asynchronous-communication.md
├── 3-data-architecture.md
├── 3.1-database-per-service.md
├── 3.2-data-flow-and-consistency.md
├── 4-infrastructure-and-deployment.md
├── 4.1-container-orchestration.md
├── 4.2-service-mesh-and-networking.md
├── 4.3-ci-cd-pipeline.md
├── 5-cross-cutting-concerns.md
├── 5.1-authentication-and-authorization.md
├── 5.2-observability.md
└── 5.3-error-handling-and-resilience.md
```

ファイル名規則: `<番号>-<kebab-case-title>.md`

**index.md に含める内容**:
1. システム名・概要（2-3段落）
2. 技術スタック全体テーブル
3. 全体アーキテクチャの Mermaid ダイアグラム（最上位俯瞰図）
4. 全セクション・ページへのリンク付き目次

---

## GitHubリポジトリ / 複数リポジトリの場合

### 単一 GitHub リポジトリ（モノレポ）
```bash
git clone --depth 1 <URL> /tmp/arch-wiki-<repo-name>
# 上記の Phase 1-5 を実行
```

### 複数リポジトリ（分散）
```bash
# 各リポジトリをサービス名ディレクトリにクローン
for svc in user-service order-service notification-service; do
  git clone --depth 1 <org-url>/$svc /tmp/arch-wiki/$svc
done
# collect_infra.sh を各リポジトリに対して実行し、出力を結合して分析
bash scripts/collect_infra.sh /tmp/arch-wiki/user-service > /tmp/infra-user.txt
bash scripts/collect_infra.sh /tmp/arch-wiki/order-service > /tmp/infra-order.txt
# ...各ファイルを順次読んで Phase 2 のアーキテクチャ分析に進む
```

---

## ページ品質基準

### importance 別の最低要件

| importance | 語数 | Mermaid | 種類 | スニペット | Sources行 | テーブル |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| high | **1200語以上** | 2-3個 | **2種類以上** | **5-8個** | 全セクション | **1個以上** |
| medium | **600-1000語** | 1-2個 | 1種類以上 | **3-5個** | 全セクション | 推奨 |
| low | 300-500語 | 1個 | 1種類以上 | 1-2個 | 主要セクション | 任意 |

### よくある失敗パターンと対策

| 失敗パターン | 対策 |
| :--- | :--- |
| ダイアグラムのサービス名が汎用的（「Service A」等） | collect_infra.sh の出力から実際のサービス名を使う |
| サービス間の通信が矢印のみで通信プロトコルが不明 | 矢印ラベルに REST/gRPC/Kafka等を明記する |
| DBの共有・専有が不明 | docker-compose の環境変数で接続先ホストを比較する |
| コードスニペットが設定ファイルではなく疑似コード | docker-compose/k8s/OpenAPIから直接引用する |
| 個別サービスの内部実装を書いてしまう | 「サービス間」の繋ぎ方に限定する |
| ページ数が少なすぎる（8ページ以下） | Service Communicationは通信方式ごとにページを分割する |
