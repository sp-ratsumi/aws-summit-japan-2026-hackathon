# Components — アフターファイブ

**Version**: 1.0
**Last Updated**: 2026-05-07
**Source of Truth**: Requirements (FR-01〜06), User Stories (US-XX), Application Design Plan
**Granularity**: 細粒度 (A1=A)

---

## Legend

- **Scope**: `frontend` / `backend` / `infra` / `cross-cutting`
- **Stories**: カバーする User Story の ID
- **FRs**: 対応する Functional Requirement の ID
- **Security Rules**: 主に適用される Security Baseline ルール
- **PBT Rules**: 主に適用される Property-Based Testing ルール

---

# Frontend Components

## FE-01: AppShell

- **Scope**: frontend
- **Responsibilities**:
  - アプリ全体のレイアウト / ルーティング / グローバルコンテキスト
  - PlatformAdapter の初期化 (Tauri / Web 判定)
  - 認証状態の初期読み込みと未認証時のログイン画面リダイレクト
- **Stories**: US-01-02
- **FRs**: FR-06-01
- **Collaborators**: PlatformAdapter (FE-09), ApiClient (FE-08), OnboardingModule (FE-02) or ReminderFeedModule (FE-03)
- **Security Rules**: SEC-08 (認証状態に応じた画面遷移)
- **Notes**: Clean Architecture の Framework & Drivers 層の入口

## FE-02: OnboardingModule

- **Scope**: frontend
- **Responsibilities**:
  - サインアップ / ログイン / パスワードリセット UI (Cognito Hosted UI ではなくカスタム)
  - 初回ヒアリング (10+ 問) の フォーム / 進行状態管理
  - 定時設定 UI
  - プロファイル編集 UI (Full)
- **Stories**: US-01-01, US-01-02, US-01-03, US-01-04, US-01-05
- **FRs**: FR-01-01, FR-01-02, FR-01-03
- **Collaborators**: ApiClient (FE-08), ProfileRepository (BE-02) via API
- **Security Rules**: SEC-04 (CSP 準拠), SEC-05 (クライアント側の一次検証), SEC-12 (パスワード要件)
- **PBT Rules**: PBT-02 (プロファイル往復), PBT-04 (冪等送信)

## FE-03: ReminderFeedModule

- **Scope**: frontend
- **Responsibilities**:
  - コンテンツカードの表示 (アプリ内モーダル / トースト / フィード)
  - Action ボタン (見た / あと少し / 興味なし / もう帰る) の受付
  - 17:00〜18:00 の頻度レンダリング (CSS アニメ、煽り演出)
- **Stories**: US-02-04, US-02-05, US-02-06, US-02-07, US-02-08, US-02-09
- **FRs**: FR-02-04, FR-03-01, FR-03-02
- **Collaborators**: SchedulerClient (FE-07), NotificationAdapter (FE-04), ApiClient (FE-08)
- **Security Rules**: SEC-05 (AI 生成テキストのエスケープ)

## FE-04: NotificationAdapter (Adapter Interface)

- **Scope**: frontend
- **Responsibilities**:
  - 通知 API の抽象 port (送信 / 権限要求)
  - 実装: `TauriNotificationAdapter` (`tauri-plugin-notification`) / `WebNotificationAdapter` (Web Notifications API)
- **Stories**: US-02-04, US-03-01
- **FRs**: FR-02-04, FR-06-01
- **Collaborators**: ReminderFeedModule (FE-03), TerminationOverlayModule (FE-05)
- **Security Rules**: SEC-04 (クリック後の画面遷移時も CSP 遵守)
- **Notes**: A3=A に基づき Adapter として実装。アプリ前面/背面の判定 (`visibilitychange`, Tauri `WindowEvent`) で適切な実装を選ぶ

## FE-05: TerminationOverlayModule

- **Scope**: frontend
- **Responsibilities**:
  - 定時到達・早期退勤トリガーでの全画面オーバーレイ表示
  - ダミー勤怠画面の描画 (退勤ボタン)
  - SE / アニメーション (ESC 無効化)
- **Stories**: US-03-01, US-03-02, US-03-03, US-03-04
- **FRs**: FR-04-01
- **Collaborators**: SchedulerClient (FE-07), ApiClient (FE-08), NotificationAdapter (FE-04)
- **Security Rules**: SEC-13 (SE 素材に SRI ハッシュ)

## FE-06: PhotoManagerModule

- **Scope**: frontend
- **Responsibilities**:
  - 画像アップロード UI (drag & drop, タグ選択)
  - 自分の画像一覧 / 削除
- **Stories**: US-04-01, US-04-03, US-04-04
- **FRs**: FR-05-01
- **Collaborators**: PlatformAdapter (FE-09) (ファイル読み込み), ApiClient (FE-08)
- **Security Rules**: SEC-05 (MIME / 拡張子 / サイズ検証), SEC-08 (自分の画像のみ表示)

## FE-07: SchedulerClient

- **Scope**: frontend
- **Responsibilities**:
  - 定時1時間前からのコンテンツ提示タイマー管理 (クライアントサイド)
  - 定時到達検知 → TerminationOverlay 起動
  - 提示レート (17:00→15min、17:40→3min 等) の計算
  - 早期退勤イベントの発火 (ReminderFeedModule の "もう帰る" ボタンから)
- **Stories**: US-02-01, US-03-01, US-03-03
- **FRs**: FR-02-01, FR-04-01
- **Collaborators**: ReminderFeedModule (FE-03), TerminationOverlayModule (FE-05), ApiClient (FE-08)
- **PBT Rules**: PBT-03 (レート invariant), PBT-05 (reference table oracle), PBT-06 (stateful)
- **Notes**: C3=A によりクライアントサイド実装。アプリ非起動時は機能しない制約を明記。

## FE-08: ApiClient

- **Scope**: frontend
- **Responsibilities**:
  - HTTPS リクエストの統一インターフェイス (Cognito JWT 自動添付、リトライ、エラー整形)
  - OpenAPI から生成した TypeScript 型で保護
- **Stories**: ALL (fe 全 module)
- **FRs**: 横断
- **Security Rules**: SEC-08 (JWT validation 前提), SEC-15 (エラーハンドリング)

## FE-09: PlatformAdapter

- **Scope**: frontend
- **Responsibilities**:
  - Geolocation / File IO / Notification 判定 / Window 制御 の抽象 port
  - 実装: `TauriPlatformAdapter` (Rust bridge 経由) / `WebPlatformAdapter` (Web API)
- **Stories**: US-02-02, US-04-01, US-05-01
- **FRs**: FR-02-03, FR-06-01
- **Security Rules**: SEC-04 (Web 版の CSP が Adapter 経由呼び出しを許容), NFR-08 (位置情報粒度)

---

# Backend Components (Python Lambda)

各バックエンドコンポーネントは論理的には独立しているが、物理的には **API Gateway のルートごとに 1 Lambda 関数** に集約され、その Lambda 内部で service 関数として呼び出される (C1=C)。監査・通知など副作用系のみ EventBridge 経由で別 Lambda に委譲 (D1=D)。

## BE-01: AuthComponent

- **Scope**: backend
- **Responsibilities**:
  - Cognito JWT の検証 (署名 / exp / aud / iss)
  - `user_id = token.sub` の抽出
  - Lambda Authorizer 関数 / 共通ミドルウェアの提供
  - IDOR チェックユーティリティ (`verify_ownership(resource_user_id, caller_user_id)`)
- **Stories**: US-01-02, US-01-05, US-02-05, US-02-09, US-04-01, US-04-02, US-04-03
- **FRs**: FR-01-01, 横断
- **Security Rules**: SEC-08, SEC-12
- **PBT Rules**: PBT-02 (JWT claim 往復)

## BE-02: ProfileComponent

- **Scope**: backend
- **Responsibilities**:
  - UserProfile CRUD (ヒアリング結果 + 定時 + 趣味 / 家族 / 推し 等)
  - 部分更新の冪等保証
- **Stories**: US-01-03, US-01-04, US-01-05
- **FRs**: FR-01-02, FR-01-03
- **Collaborators**: RepositoryLayer (`ProfileRepository`), AuthComponent (BE-01), AuditComponent (BE-11)
- **Security Rules**: SEC-05 (入力検証), SEC-08 (所有権), SEC-13 (プロファイル変更の監査)
- **PBT Rules**: PBT-02 (round-trip), PBT-04 (idempotent), PBT-07 (domain generator)

## BE-03: SchedulerComponent (Backend 側)

- **Scope**: backend
- **Responsibilities**:
  - 「次回提示時刻」計算ロジックの **参照実装** (reference table) を提供 (PBT-05 oracle 用)
  - クライアントから提示イベントを受けて履歴に書き込むエンドポイント (ContentSelection から呼び出される)
- **Stories**: US-02-01
- **FRs**: FR-02-01
- **Collaborators**: HistoryComponent (BE-07)
- **PBT Rules**: PBT-03 (invariant), PBT-05 (oracle)
- **Notes**: クライアントサイドスケジューラ (FE-07) のオラクルとして機能、同一ロジックを重複実装する

## BE-04: ContentSelectionComponent

- **Scope**: backend
- **Responsibilities**:
  - Amazon Bedrock (Claude) へのプロンプト生成 + 呼び出し + 応答パース
  - タイムアウト / 例外時のフォールバック (テンプレート文言 + ルールベース選定)
  - 履歴重複排除 / カテゴリ多様性の invariant 保証
  - 入力: `{ userId, profile, context(time/location/weather), recentHistory(N=10) }`
  - 出力: `{ category, contentId, title, body, imageUrl? }`
- **Stories**: US-02-03, US-02-06, US-02-07, US-02-08, US-03-04, US-04-02
- **FRs**: FR-02-01, FR-02-02
- **Collaborators**: ContentRepositoryComponent (BE-05), PhotoComponent (BE-06), HistoryComponent (BE-07), Bedrock (external)
- **Security Rules**: SEC-03 (PII 除外ログ), SEC-05 (プロンプト長検証), SEC-15 (fail-safe default)
- **PBT Rules**: PBT-03 (多様性 invariant)

## BE-05: ContentRepositoryComponent

- **Scope**: backend
- **Responsibilities**:
  - ダミーデータ (飲食 / 趣味 / 電車 / 天気) の検索 (`findByCategoryAndLocation`, `findByHobbyCategory`, etc.)
  - DynamoDB の Content テーブル (単一テーブル設計の `Content#<category>` prefix) をラップ
- **Stories**: US-02-06, US-02-07, US-02-08
- **FRs**: FR-02-02
- **Security Rules**: SEC-01 (DynamoDB 暗号化), SEC-05 (クエリパラメータ検証)

## BE-06: PhotoComponent

- **Scope**: backend
- **Responsibilities**:
  - S3 Pre-signed URL 発行 (PUT, `user/{userId}/photos/*` に限定)
  - メタデータの CRUD (PhotoTable)
  - 削除 (S3 delete + メタ delete)
- **Stories**: US-04-01, US-04-03, US-04-04
- **FRs**: FR-05-01
- **Collaborators**: AuthComponent (BE-01), ContentSelectionComponent (BE-04) (family/pet 写真選択), AuditComponent (BE-11)
- **Security Rules**: SEC-01, SEC-05, SEC-08 (IDOR), SEC-09 (バケット Public Block), SEC-13 (削除監査)
- **PBT Rules**: PBT-02 (メタ情報往復)

## BE-07: HistoryComponent

- **Scope**: backend
- **Responsibilities**:
  - 提示履歴の append (userId, timestamp, category, contentId, ...)
  - 指定日の履歴取得 / 直近 N 件取得
- **Stories**: US-02-01, US-02-03, US-02-09
- **FRs**: FR-03-01
- **Security Rules**: SEC-08 (自分の履歴のみ), SEC-14 (ログ保持 ≥ 90 日 相当の DynamoDB TTL)

## BE-08: ReactionComponent

- **Scope**: backend
- **Responsibilities**:
  - リアクション (SEEN / IGNORE / LEAVE_NOW) の永続化
  - 冪等 key (`userId#contentId#action#timestamp`) による重複排除
  - 「もう帰る」アクション → TerminationComponent へイベント発火
- **Stories**: US-02-05, US-03-03
- **FRs**: FR-03-02
- **Collaborators**: TerminationComponent (BE-09) (EventBridge 経由, D1=D)
- **Security Rules**: SEC-08
- **PBT Rules**: PBT-04 (idempotent)

## BE-09: TerminationComponent

- **Scope**: backend
- **Responsibilities**:
  - 退勤イベントの記録 (日次一意、同日再発火しない)
  - ねぎらい一言生成 (Bedrock 呼び出し、フォールバックあり)
- **Stories**: US-03-02, US-03-03, US-03-04
- **FRs**: FR-04-01
- **Collaborators**: ContentSelectionComponent (BE-04) のプロンプト再利用, AuditComponent (BE-11)
- **Security Rules**: SEC-14 (退勤記録の監査), SEC-15 (fail-safe default)

## BE-10: NotificationComponent (Server Side, Optional MVP)

- **Scope**: backend
- **Responsibilities**:
  - (Full 向け) Web Push 購読情報の管理、クライアントへのプッシュ配信
  - Tauri 向けには BFF 不要 (クライアントが OS 通知を直接叩く)
- **Stories**: 主に Full (将来拡張)
- **FRs**: FR-02-04 の一部
- **Security Rules**: SEC-08 (購読は本人のみ), SEC-13 (VAPID 鍵は Secrets Manager)
- **Notes**: MVP では最小 (push 配信は実装しない、API のスキーマのみ定義)

## BE-11: AuditComponent

- **Scope**: cross-cutting (backend)
- **Responsibilities**:
  - 重要操作 (profile 更新 / photo 削除 / アカウント削除) の監査ログ書き込み
  - append-only DynamoDB テーブル + CloudWatch Logs (2 系統で冗長)
  - `before`, `after`, `actor`, `timestamp`, `source_ip` を記録
- **Stories**: US-04-03, US-05-02, US-05-05, US-03-03
- **FRs**: 横断 (NFR-03)
- **Security Rules**: SEC-03, SEC-13, SEC-14

## BE-12: InfrastructureComponent (IaC)

- **Scope**: infra
- **Responsibilities**:
  - AWS CDK (Python) で全リソースをコード化
  - Stacks: NetworkStack / AuthStack / DataStack / ApiStack / FrontendStack / ObservabilityStack
- **Stories**: 横断
- **FRs**: NFR-02, NFR-03, NFR-07
- **Security Rules**: SEC-01, SEC-02, SEC-04, SEC-06, SEC-07, SEC-09, SEC-14 (インフラ層で担保)

---

# Cross-Cutting / Shared Modules

## X-01: Observability (Lambda Powertools Logger / Tracer / Metrics)

- **Scope**: cross-cutting
- **E3=A 準拠**
- 構造化ログ (correlation id、X-Ray trace id)
- PII filter を middle として挿入
- **Security Rules**: SEC-03, SEC-14

## X-02: SharedModels (Pydantic + OpenAPI)

- **Scope**: cross-cutting
- **Responsibilities**:
  - バックエンドの Pydantic モデル (入力検証 + シリアライズ)
  - OpenAPI YAML (`openapi/schema.yaml`) を single source of truth に
  - `openapi-typescript` でフロントの TS 型を生成
- **Security Rules**: SEC-05 (入力検証の実装)
- **PBT Rules**: PBT-02 (Model ↔ JSON round-trip)

## X-03: RepositoryLayer (DynamoDB Single-table)

- **Scope**: cross-cutting
- **D2=C 準拠**: single-table + Repository 抽象
- テーブル: `AfterFiveTable` (PK=partitionKey, SK=sortKey)
  - `USER#<userId>` / `PROFILE` → ユーザープロファイル
  - `USER#<userId>` / `PHOTO#<photoId>` → 写真メタ
  - `USER#<userId>` / `HISTORY#<iso-ts>#<contentId>` → 提示履歴
  - `USER#<userId>` / `REACTION#<iso-ts>#<contentId>` → リアクション
  - `USER#<userId>` / `TERMINATION#<yyyy-mm-dd>` → 退勤記録 (日次一意)
  - `CONTENT#<category>` / `<contentId>` → ダミーコンテンツ
  - `AUDIT#<yyyy-mm-dd>` / `<iso-ts>#<actor>` → 監査ログ (別テーブル案も)
- Repository Classes: `ProfileRepository`, `PhotoRepository`, `HistoryRepository`, `ReactionRepository`, `TerminationRepository`, `ContentRepository`, `AuditRepository`
- **PBT Rules**: PBT-02 (serialize/deserialize), PBT-04 (idempotent put)

## X-04: ErrorHandling (例外階層)

- **Scope**: cross-cutting (backend)
- **B2=B 準拠**
- Custom exception hierarchy:
  - `AppError` (基底)
    - `ValidationError` → 400
    - `AuthenticationError` → 401
    - `AuthorizationError` → 403 (IDOR 時)
    - `NotFoundError` → 404
    - `ConflictError` → 409 (重複)
    - `UpstreamError` → 502 (Bedrock 失敗等)
    - `InternalError` → 500
- 全エンドポイントの top-level でキャッチ、汎用メッセージで返す (SEC-09)
- **Security Rules**: SEC-15

---

# Component Count Summary

| 区分 | Count |
|---|---:|
| Frontend | 9 (FE-01〜FE-09) |
| Backend | 12 (BE-01〜BE-12) |
| Cross-Cutting | 4 (X-01〜X-04) |
| **Total** | **25** |

(A1=A 細粒度方針に沿う。1 責務 1 コンポーネントを維持)

---

# Component Traceability Matrix (Story → Component)

| Story | Primary Components |
|---|---|
| US-01-01 ユーザー登録 | FE-02, BE-01, BE-02, X-02 |
| US-01-02 ログイン | FE-01, FE-02, FE-08, BE-01 |
| US-01-03 ヒアリング | FE-02, BE-02, X-02, X-03 |
| US-01-04 定時設定 | FE-02, BE-02, X-03 |
| US-01-05 プロファイル編集 | FE-02, BE-01, BE-02, BE-11 |
| US-02-01 スケジューラ | FE-07, BE-03, BE-07 |
| US-02-02 コンテキスト収集 | FE-09, FE-08 |
| US-02-03 AI 選定 | FE-07, BE-04, BE-05, BE-07, Bedrock |
| US-02-04 提示 UI | FE-03, FE-04 |
| US-02-05 リアクション | FE-03, BE-01, BE-08 |
| US-02-06 飲食店 | BE-04, BE-05 |
| US-02-07 趣味系 | BE-04, BE-05 |
| US-02-08 電車・天気 | BE-04, BE-05 |
| US-02-09 履歴 | FE-03, BE-07 |
| US-03-01 定時オーバーレイ | FE-05, FE-07, FE-04 |
| US-03-02 ダミー勤怠 | FE-05, BE-09 |
| US-03-03 早期退勤 | FE-03, FE-05, BE-08, BE-09 |
| US-03-04 退勤メッセージ | FE-05, BE-09, BE-04 |
| US-04-01 写真アップロード | FE-06, FE-09, BE-01, BE-06 |
| US-04-02 AI キャプション | BE-04, BE-06 |
| US-04-03 写真削除 | FE-06, BE-06, BE-11 |
| US-04-04 タグ分類 | FE-06, BE-06 |
| US-05-01 位置情報粒度 | FE-09 |
| US-05-02 アカウント削除 | FE-02, BE-01, BE-02, BE-06, BE-11 |
| US-05-03 セキュリティヘッダー | BE-12 (CloudFront Response Headers Policy) |
| US-05-04 レート制限 | BE-12 (API Gateway Usage Plan) |
| US-05-05 監査ログ | BE-11, BE-12 |

全 27 Story にマッピングあり ✅
