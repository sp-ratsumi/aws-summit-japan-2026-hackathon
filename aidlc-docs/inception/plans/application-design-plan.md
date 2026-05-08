# Application Design Plan — アフターファイブ

**Stage**: INCEPTION — Application Design
**Purpose**: 高レベルコンポーネント・サービス層・依存関係の設計計画 (詳細ビジネスロジックは後の Functional Design で)

---

## 1. 前提コンテキスト

- **Project Type**: Greenfield (Tauri デスクトップ + Web PWA + サーバーレス AWS)
- **参照ドキュメント**:
  - `aidlc-docs/inception/requirements/requirements.md` (FR-01〜06, NFR-01〜08)
  - `aidlc-docs/inception/user-stories/stories.md` (27 Stories / 5 Epics / 4 Personas)
- **主要アーキテクチャ方針**:
  - Frontend: React + TypeScript + Vite (同一コードベースから Tauri と Web PWA の 2 ビルド)
  - Backend: Python on AWS Lambda + API Gateway + Cognito + DynamoDB + S3 + CloudFront + Bedrock (Claude)
  - 拡張: Security Baseline (Full) + Property-Based Testing (Full)

---

## 2. Application Design Execution Checklist

以下の順で実行予定 (現時点は Planning):

- [ ] Step A: `components.md` 生成 — コンポーネント定義と高レベル責務
- [ ] Step B: `component-methods.md` 生成 — 各コンポーネントのメソッドシグネチャ (詳細ビジネスルールは Functional Design で)
- [ ] Step C: `services.md` 生成 — サービス層定義とオーケストレーションパターン
- [ ] Step D: `component-dependency.md` 生成 — 依存関係マトリクスと通信パターン
- [ ] Step E: `application-design.md` 生成 — 上記を統合した包括的設計書

---

## 3. Pre-Proposed Architecture (提案)

User Stories から抽出したコンポーネント候補 (初期案) — 回答に応じて調整します:

### Frontend (React + TypeScript)
- **AppShell** — ルーティング / レイアウト / アプリ初期化
- **OnboardingModule** — サインアップ / ログイン / 初回ヒアリング (ダメな欲望プロファイル) / 定時設定 (ダメモード突入時刻)
- **ReminderFeedModule** — ダメな未来コンテンツ提示 UI、アプリ内モーダル / トースト
- **NotificationAdapter** — プラットフォーム差分吸収 (Tauri OS 通知 / Web Notifications API)
- **TerminationOverlayModule** — 堕落ゲート (全画面オーバーレイ) + ダミー勤怠画面 + 退勤 SE
- **PhotoManagerModule** — 写真アップロード UI (D1 ダメ欲望素材) / 一覧 / 削除
- **SchedulerClient** — クライアントサイドで 堕落ランプ を刻む + 可視性検知
- **ApiClient** — Cognito JWT 添付・リトライ・エラー整形
- **PlatformAdapter** — Geolocation / FileSystem / Notification の Tauri/Web 差分

### Backend (Python Lambda)
- **AuthComponent** — Cognito JWT 検証 / 認可 (IDOR チェック)
- **ProfileComponent** — ダメな欲望プロファイル CRUD (ヒアリング + 定時 + 編集)
- **SchedulerComponent** — 堕落ランプ レート計算 / 次回提示時刻算出 (参照実装含む)
- **ContentSelectionComponent** (= ダメな未来ジェネレータ) — Bedrock プロンプト生成 / 応答整形 / フォールバック / D4「飲まない」除外
- **ContentRepositoryComponent** — ダミーデータ (飲食 / 趣味 / 電車 / 天気 / 酒) 管理
- **PhotoComponent** — Pre-signed URL 発行 / メタ CRUD / 削除 / IDOR チェック
- **HistoryComponent** — ダメ堕ち履歴の永続化 / 読取
- **ReactionComponent** — リアクション記録 (もう堕ちる含む) / 冪等書込 / カテゴリ重み学習フック
- **TerminationComponent** — 堕落ゲート発動記録 / ダメモード突入メッセージ生成
- **NotificationComponent** — Web Push 登録 / 送信 (オプション、MVP 最小)
- **AuditComponent** — 監査ログ write-only
- **InfrastructureComponent** (IaC) — CDK/SAM による AWS リソース定義

### Service Layer
- **ReminderOrchestrationService** — 堕落ランプ → ダメな未来ジェネレータ → Content/Photo → History を統合
- **OnboardingOrchestrationService** — Signup → Hearing (ダメな欲望プロファイル) → Profile 保存までの flow
- **TerminationOrchestrationService** — 定時到達 → 堕落ゲート発動 → 退勤記録 → ダメモード突入メッセージ

---

## 4. Questions for Application Design Planning

**回答方法**: 各質問の `[Answer]:` タグに A/B/C/D/X を記入してください。

### A. Component Identification (コンポーネント識別)

### Question A1: バックエンドコンポーネント粒度
バックエンドコンポーネントの分割粒度は?

A) **細粒度 (12+ コンポーネント)**: 上記の初期案通り、1 責務 1 コンポーネント。Units 分解が綺麗、テスト容易
B) **中粒度 (7-8 コンポーネント)**: 認可・プロファイル系を統合、コンテンツ系を統合。バランス型
C) **粗粒度 (4-5 コンポーネント)**: 機能ドメイン単位で大きくまとめる (Identity / Reminder / Photo / Termination / Infrastructure)、シンプル
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question A2: フロントエンドのアーキテクチャパターン
フロントエンドで採用するアーキテクチャパターンは?

A) **Feature-Sliced Design** (features/、entities/、shared/ の階層)
B) **古典的な構成** (components/、pages/、hooks/、services/ のフラット構成)
C) **Clean Architecture (Ports & Adapters)** — プラットフォーム差分吸収 (Tauri/Web) に適する
D) **Atomic Design + Hooks** (atoms/molecules/organisms/templates/pages)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question A3: プラットフォーム差分 (Tauri / Web) の扱い
Tauri と Web の差分を吸収する方針は?

A) **Adapter パターン** — `NotificationAdapter`, `LocationAdapter`, `FileAdapter` などを interface として定義し、Tauri / Web で別実装を差し替え
B) **条件分岐** — `if (window.__TAURI__)` 等でコード内分岐
C) **完全別ビルド** — フロントから下の層 (platform/) を 2 つ用意、上位 UI だけ共通
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### B. Component Methods

### Question B1: メソッドシグネチャの記述スタイル
`component-methods.md` のメソッドシグネチャは?

A) **TypeScript / Python の型付き疑似コード** (実装に近い具象)
B) **OpenAPI / JSON Schema 風の抽象仕様** (入出力型 + セマンティクス、実装言語依存なし)
C) **併用**: バックエンド API は OpenAPI、内部メソッドは言語別の型付きコード
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question B2: エラー戦略
エラーハンドリング / エラー型の方針は?

A) **Result/Either 型** (例外を使わない、関数型アプローチ)
B) **例外ベース** (Python: カスタム例外階層、TypeScript: `throw new Error(...)`)
C) **ハイブリッド**: バックエンドは例外、フロントは Result 型
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### C. Service Layer

### Question C1: サービスレイヤの実装形態
バックエンドのサービスレイヤは?

A) **Orchestration Lambda Functions** — 複合フロー (Onboarding / Reminder / Termination) ごとに独立した Lambda + Step Functions なし
B) **AWS Step Functions** で state machine として定義 — 可観測性高い、コスト若干増
C) **単一 Lambda 内の service 関数** — API Gateway のルートごとに Lambda、その中にサービス層関数を配置
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question C2: 非同期処理の扱い
Bedrock 呼び出しや画像処理のような遅延しうる処理は?

A) **同期 (API Gateway → Lambda → Bedrock → レスポンス)** 最大 10 秒タイムアウト
B) **非同期 (API → SQS → Worker Lambda)**、結果はクライアントがポーリング or WebSocket
C) **ハイブリッド**: MVP は同期、Bedrock 負荷が増えたら非同期に移行できる設計
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question C3: スケジューラの実行場所
コンテンツ提示のトリガー (動的頻度) は?

A) **クライアントサイド** (Tauri/Web で setTimeout/setInterval) — API コストゼロ、ただしアプリ非起動時は動かない
B) **EventBridge Scheduler (サーバーサイド)** — ユーザーごとに cron ルールを動的生成、Lambda をトリガー
C) **Step Functions Wait + Loop** — 状態機械でユーザーセッションを管理
D) **ハイブリッド**: 起動中はクライアント、バックグラウンド時のみサーバー側プッシュ
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### D. Dependencies / Communication

### Question D1: コンポーネント間通信パターン
バックエンドのコンポーネント同士の通信は?

A) **直接呼び出し (同一プロセス内関数呼び出し)** — モジュール分離は論理のみ、物理的には monolithic Lambda
B) **Lambda → Lambda invoke (sync)** — 各コンポーネントが独立 Lambda、互いに InvocationType=Sync
C) **イベント駆動 (EventBridge / SNS)** — 疎結合、監視容易
D) **ハイブリッド**: 主要フローは A、監査や通知のような副作用は C
X) Other (please describe after [Answer]: tag below)

[Answer]: D

### Question D2: データアクセスパターン
DynamoDB へのアクセスは?

A) **Repository パターン** — 各コンポーネントが Repository クラス経由で DynamoDB にアクセス、テストでは mock
B) **直接 boto3 呼び出し** — シンプル、抽象化薄い
C) **Single-table design** with Repository (1 テーブル + Repository 抽象)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question D3: フロント/バック間の API 契約管理
API 契約はどう管理しますか?

A) **OpenAPI Spec (YAML)** を single source of truth に、フロント/バック両方で型生成 (`openapi-typescript` 等)
B) **tRPC / GraphQL** で型駆動
C) **手動**: バックエンドの Pydantic モデル と フロントの TypeScript 型を手で合わせる
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### E. Design Patterns

### Question E1: 全体のアーキテクチャスタイル
全体のアーキテクチャスタイルは?

A) **Clean Architecture (Hexagonal / Ports & Adapters)** — ドメイン中心、I/O は adapter で
B) **Domain-Driven Design (DDD) Lite** — Bounded Context を意識しつつ軽量
C) **実用的なレイヤードアーキテクチャ** (controller → service → repository) — シンプル・手堅い
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question E2: 認可 (IDOR 対策) の実装箇所
Security Baseline (SECURITY-08) の Object-Level Authorization はどこで実装しますか?

A) **API Gateway + Lambda Authorizer** で JWT 検証 + リソースパス からの userId チェック
B) **各 Lambda の冒頭で共通ミドルウェア** (`@requires_ownership(resource='photo', id_param='photoId')`)
C) **Repository 層で** userId を必ず where 条件に混ぜる (DynamoDB partition key を userId にする設計)
D) **多層防御**: A + B + C を全部 (推奨、Defense in Depth)
X) Other (please describe after [Answer]: tag below)

[Answer]: D

### Question E3: ログ / 監査の統一パターン
ログ/監査書き込みの方針は?

A) **AWS Lambda Powertools for Python** (Logger / Tracer / Metrics 統一) + CloudWatch Logs + 監査用別 DynamoDB テーブル
B) **OpenTelemetry** + CloudWatch
C) **print() 中心 + 手動 JSON** (シンプル、MVP 向け)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 5. 回答後の Generation ステップ (未着手)

以下は回答承認後に実行:

- [ ] Step A: `aidlc-docs/inception/application-design/components.md` 生成
- [ ] Step B: `aidlc-docs/inception/application-design/component-methods.md` 生成
- [ ] Step C: `aidlc-docs/inception/application-design/services.md` 生成
- [ ] Step D: `aidlc-docs/inception/application-design/component-dependency.md` 生成 (依存マトリクス + 通信パターン + 簡潔なデータフロー図)
- [ ] Step E: `aidlc-docs/inception/application-design/application-design.md` 生成 (上記統合)
- [ ] Step F: Security Baseline / PBT の適用方針を各コンポーネントに反映
- [ ] Step G: Self-review (責務分離 / 循環依存なし / Story → Component マッピング完全性)
