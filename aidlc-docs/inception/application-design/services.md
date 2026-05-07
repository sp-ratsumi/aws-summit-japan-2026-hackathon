# Services — アフターファイブ

**Version**: 1.0
**Service Layer Style**: 単一 Lambda 内の service 関数 (C1=C)、orchestrator は機能ドメイン単位の関数として配置。副作用 (監査/通知) は EventBridge 経由 (D1=D)
**Overall Architecture**: 実用的レイヤードアーキテクチャ (E1=C): controller → service → repository → data

---

## Service Layer の位置付け

```
┌─────────────────────────────────────────────────────┐
│ Controller (Lambda handler)                          │
│  - 入力バリデーション (Pydantic)                     │
│  - 認可 (AuthComponent middleware)                   │
│  - エラー handling (X-04)                             │
└──────────────┬──────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│ Service (このドキュメントの対象)                     │
│  - 複数コンポーネント/リポジトリの協調               │
│  - ビジネストランザクション境界                     │
│  - 冪等化 / トランザクション (DynamoDB TransactWrite) │
└──────────────┬──────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│ Repository (X-03, DynamoDB / S3 を抽象化)            │
└─────────────────────────────────────────────────────┘
```

---

# S-01: OnboardingOrchestrationService

**Responsibilities**: 新規ユーザーがサインアップ〜ヒアリング完了までを繋ぐ一連の協調

**Orchestrates**: BE-01 AuthComponent、BE-02 ProfileComponent、BE-11 AuditComponent (via EventBridge)

## Operations

### `complete_signup_and_hearing(jwt: str, hearing_answers: HearingAnswers) → UserProfile`

**Flow**:
1. `AuthComponent.verify_jwt(jwt)` → JwtClaims (fail: AuthenticationError)
2. `ProfileComponent.put_initial_profile(user_id, hearing_answers)` (冪等)
3. EventBridge emit `user.onboarded` イベント (AuditComponent が consume)
4. 結果の UserProfile を返却

**Invariants** (PBT):
- 同一 jwt + 同一 hearing_answers の重複呼び出しは同一結果を返す (PBT-04)
- 失敗時は DynamoDB 状態が変更されていない

**Security**: SEC-05 (入力検証は Controller で Pydantic、Service で不変条件チェック), SEC-03 (ログに PII 含めない)

---

# S-02: ReminderOrchestrationService (最重要)

**Responsibilities**: クライアントスケジューラが「次のコンテンツちょうだい」と叩く主フローを統括

**Orchestrates**: BE-01 AuthComponent、BE-02 ProfileComponent (read)、BE-07 HistoryComponent (read + append)、BE-04 ContentSelectionComponent、BE-05 ContentRepositoryComponent、BE-06 PhotoComponent、External Bedrock

## Operations

### `select_and_record_next_content(user_id: str, context: ReminderContext) → ContentResponse`

**Flow**:
1. Parallel read:
   - `ProfileRepository.get(user_id)` → UserProfile
   - `HistoryRepository.get_recent(user_id, n=10)` → RecentHistory
2. Build prompt:
   - `ContentSelectionComponent.build_prompt(profile, context, recent_history)` (PII stripped)
3. Invoke Bedrock with 5s timeout:
   - 成功 → parse, sanitize, validate (category ∈ allowed, body length ∈ [20,80])
   - 失敗/タイムアウト → fallback:
     - カテゴリを profile の hobby から重み付きランダム選定
     - ContentRepository.find_* で候補取得
     - テンプレート文言 (`「{category}が待ってるよ」`) で body 生成
4. カテゴリが `family|pet` なら PhotoRepository から 1 枚選んで imageUrl に埋め込み
5. HistoryComponent.append(user_id, contentId, category, body) (idempotent append)
6. ContentResponse 返却

**Invariants** (PBT):
- 同一 context で 10 回呼び出したとき、同一カテゴリが 3 連続以上発生しない (PBT-03)
- body 長さは常に 20〜80 chars (PBT-03)
- failure path でも response は必ず返る (SEC-15 fail-safe)

**Latency Target**: p95 3 秒、p99 5 秒 (Bedrock タイムアウト 5 秒)

**Security**: SEC-03 (PII 除外), SEC-08 (user_id は JWT 由来のみ、body に他人情報含まず), SEC-15 (fail-safe default)

---

# S-03: TerminationOrchestrationService

**Responsibilities**: 定時到達 / 早期退勤ボタン押下時の一連処理

**Orchestrates**: BE-01 AuthComponent、BE-09 TerminationComponent、BE-04 ContentSelectionComponent (ねぎらい生成)、BE-11 AuditComponent (via EventBridge)

## Operations

### `clock_out(user_id: str, trigger: Literal['PUNCTUALITY','EARLY'], timestamp: datetime) → TerminationResponse`

**Flow**:
1. 日付キー `yyyy-mm-dd` を timestamp から算出 (JST)
2. `TerminationRepository.put_if_absent(user_id, date_key, trigger, timestamp)` (DynamoDB `ConditionExpression: attribute_not_exists(PK)`)
   - 初回 → 2 に進む
   - 2 度目以降 → `recorded: false`, 既存 message を返却して終了
3. ねぎらい文生成:
   - `ContentSelectionComponent.generate_closing_message(profile, trigger, today_history)` 呼び出し
   - Bedrock 呼び出し → 20-50 chars
   - 失敗時: テンプレート (`「お疲れさま！明日も頑張ろう」`)
4. Message を Termination レコードに追記保存 (optional、監査上便利)
5. EventBridge emit `user.terminated` (AuditComponent consume)
6. `{ recorded: true, message }` を返却

**Invariants**:
- 同一日に何度呼ばれても DynamoDB 上の Termination レコードは 1 件 (PBT-04)
- `recorded` は初回 true、以降 false

**Security**: SEC-14 (監査イベント発火), SEC-15 (Bedrock 失敗でも処理続行)

---

# S-04: ReactionHandlerService

**Responsibilities**: ユーザーリアクション (SEEN/IGNORE/LEAVE_NOW) の受付と二次効果

**Orchestrates**: BE-01 AuthComponent、BE-08 ReactionComponent、BE-07 HistoryComponent、EventBridge

## Operations

### `record_reaction(user_id: str, req: ReactionRequest) → ReactionResponse`

**Flow**:
1. HistoryRepository で contentId の所有権を確認 (自分の履歴にあるか) → なければ AuthorizationError (IDOR)
2. ReactionRepository.put_if_absent(clientIdempotencyKey, ...) (冪等)
3. action によるディスパッチ:
   - `LEAVE_NOW` → EventBridge `reaction.leave_now_requested` emit → TerminationOrchestrationService が consumer ラムダで clock_out を先行記録 (クライアントの退勤ボタン押下待ち)
   - `IGNORE` → (Full) カテゴリ重みを減点する event emit (MVP では実装しない、永続化のみ)
   - `SEEN` → (Full) カテゴリ重みを若干加点
4. AuditEvent は LEAVE_NOW の時のみ発火 (SEEN/IGNORE は頻度が高く audit 肥大を避ける)

**Invariants**:
- 同一 clientIdempotencyKey で N 回呼ばれても DynamoDB 状態は同一 (PBT-04)

**Security**: SEC-08 (IDOR), SEC-15

---

# S-05: PhotoUploadService

**Responsibilities**: 写真アップロードの一連 (presigned URL 発行 → 確認)

**Orchestrates**: BE-01 AuthComponent、BE-06 PhotoComponent、S3 (direct)

## Operations

### `issue_upload_url(user_id: str, req: UploadUrlRequest) → UploadUrlResponse`

**Flow**:
1. Validate MIME/ext/size (Pydantic) → `ValidationError`
2. uuid 生成 → `photo_id`
3. S3 Pre-signed URL 生成 (PUT, 15min expiry, Content-Type 固定、Content-Length-Range 制限)
4. PhotoRepository.put(user_id, photo_id, metadata, status=PENDING)
5. Response 返却

**Security**: SEC-01 (S3 SSE), SEC-05 (サイズ・形式), SEC-08 (prefix 制限), SEC-09 (バケット Public Block)

### `confirm_upload(user_id: str, photo_id: str) → Photo`

**Flow**:
1. PhotoRepository.get(user_id, photo_id) → Photo (status PENDING)
2. S3 HEAD でオブジェクト存在 + サイズ再検証
3. PhotoRepository.update_status(photo_id, ACTIVE)
4. EventBridge emit `photo.uploaded` (AuditComponent consume、Full では Bedrock での AI タグ付与をキック)

### `delete_photo(user_id: str, photo_id: str) → None`

**Flow**:
1. PhotoRepository.get → IDOR チェック
2. S3 DeleteObject
3. PhotoRepository.soft_delete (status=DELETED)
4. EventBridge emit `photo.deleted` (AuditComponent consume)

**Security**: SEC-08 (IDOR multi-layer: Authorizer + Service check + Repository user_id), SEC-13 (audit)

---

# S-06: AccountDeletionService (Full)

**Responsibilities**: アカウント削除 (プロファイル + 写真 + 履歴 + リアクション + Cognito ユーザー) の包括削除

**Orchestrates**: BE-01 AuthComponent、BE-02 ProfileComponent、BE-06 PhotoComponent、BE-07 HistoryComponent、BE-08 ReactionComponent、Cognito (external)

## Operations

### `delete_account(user_id: str, password_reconfirm: str) → DeletionReceipt`

**Flow**:
1. パスワード再認証 (Cognito `AdminInitiateAuth`) — PII 削除前の最終防壁
2. EventBridge emit `account.deletion.started` (進行状況のため)
3. Async (Step Functions or SQS queue worker):
   - S3 プレフィックス `user/<user_id>/` の全オブジェクト削除 (バッチ)
   - DynamoDB `USER#<user_id>` で始まる全アイテムを Query → BatchWriteItem で削除
   - Cognito `AdminDeleteUser`
4. 完了時 EventBridge emit `account.deletion.completed` → AuditComponent が不可逆記録
5. ユーザーには即座にログアウト & 「削除を受け付けました、完了までに最大 24 時間かかります」と返却

**Invariants**:
- 開始後は stoppable ではない (PUSHED audit event を残す)
- 同一ユーザーの複数削除要求は冪等 (Step Functions の execution name を `deletion-<user_id>` にする)

**Security**: SEC-12 (password reconfirm), SEC-14 (audit)

---

# Service Orchestration Flow Diagrams

## Reminder Flow (コア体験)

```
Client (FE-07)       API Gateway          Lambda (s2_reminder)       Bedrock
    │                    │                       │                     │
    │ 17:15 timer hits   │                       │                     │
    ├──► POST            │                       │                     │
    │   /content/next    │                       │                     │
    │                    ├──► invoke             │                     │
    │                    │                       ├── verify_jwt        │
    │                    │                       ├── read profile      │
    │                    │                       ├── read history      │
    │                    │                       ├──► InvokeModel ────►│
    │                    │                       │                     ├ 3s
    │                    │                       │◄── response ────────┤
    │                    │                       ├── sanitize/validate │
    │                    │                       ├── history.append    │
    │                    │                       │◄─ response          │
    │                    │◄──────────────────────┤                     │
    │◄── 200 ContentResp │                       │                     │
    │                    │                       │                     │
    │ render modal       │                       │                     │
    │ (FE-03 feed)       │                       │                     │
```

## Termination Flow

```
Client (FE-05)       API Gateway          Lambda (s3_termination)    EventBridge
    │                    │                       │                        │
    │ 18:00 overlay      │                       │                        │
    │ button press       │                       │                        │
    ├──► POST            │                       │                        │
    │   /terminations/   │                       │                        │
    │      clock-out     │                       │                        │
    │                    ├──► invoke             │                        │
    │                    │                       ├ verify_jwt             │
    │                    │                       ├ put_if_absent date=today
    │                    │                       ├ generate_closing_msg   │
    │                    │                       │ (Bedrock or fallback)  │
    │                    │                       ├──► emit ──────────────►│
    │                    │                       │    user.terminated     │ ──► Audit Lambda
    │                    │                       │◄─ msg                  │
    │◄── 200 message     │◄──────────────────────┤                        │
```

---

# Service Boundary Rules

1. **1 Controller = 1 Lambda = 1 または小数の Service 呼び出し** (C1=C)
2. **Services は Controller からのみ呼ばれる** (Service 間呼び出しは避ける、必要なら EventBridge)
3. **Repository は Service からのみ呼ばれる** (Controller から直接 Repository を叩かない)
4. **External API (Bedrock, Cognito) は専用の Component 経由**、Service から直接 boto3 しない
5. **冪等性は Service 層で担保** (DynamoDB の ConditionExpression、クライアント idempotency key)
6. **トランザクション境界は Service 内**、DynamoDB TransactWriteItems で複数エンティティ同時更新

---

# Non-Functional Aspects (Service Level)

| 観点 | 対応 |
|---|---|
| **冪等性** | Service 層で ClientIdempotencyKey or DynamoDB ConditionExpression を使用 (PBT-04) |
| **タイムアウト** | Bedrock 5s, DynamoDB 1s, S3 3s。Service 全体 API Gateway 29s 以内 |
| **リトライ** | idempotent な write は自動リトライ (boto3 default)、非 idempotent はリトライ禁止 |
| **可観測性** | X-01 Observability: Service 開始/終了/例外を構造化ログ、correlation id 付与 |
| **テスタビリティ** | Service 関数は DI (Repository / Bedrock client を引数で受ける) で unit testable |
