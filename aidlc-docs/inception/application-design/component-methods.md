# Component Methods — アフターファイブ

**Version**: 2.0
**Last Updated**: 2026-05-08
**Method Signature Style**: OpenAPI / JSON Schema 風 (B1=B)
**Scope**: 各コンポーネントの "公開メソッド" のインターフェイス (詳細ビジネスルールは Functional Design で)

---

## Notation

- `Method`: メソッド名 (snake_case / camelCase を言語に合わせて意訳)
- `Input`: 入力スキーマ (OpenAPI 風)
- `Output`: 出力スキーマ
- `Errors`: 発生し得るエラー (X-04 ErrorHandling の階層から)
- `Side Effects`: 副作用 (DB 書き込み / EventBridge / Bedrock 呼び出し 等)

---

# Backend Component Methods (API endpoints 主体)

## BE-01: AuthComponent

### `verify_jwt(authorization_header: string) → JwtClaims`
- **Input**: `{ Authorization: Bearer <token> }` ヘッダ
- **Output**:
  ```yaml
  JwtClaims:
    sub: string (userId)
    email: string
    exp: integer
    iat: integer
  ```
- **Errors**: `AuthenticationError` (署名不正 / exp 切れ)
- **Side Effects**: なし (純粋)

### `verify_ownership(claim_user_id: string, resource_user_id: string) → void`
- **Input**: 両 userId
- **Output**: void (throw しなければ OK)
- **Errors**: `AuthorizationError` (不一致時)

### `require_auth(handler)` / `@requires_ownership(resource, id_param)` decorator
- Lambda ハンドラの前置きに適用する共通ミドルウェア
- `E2=D` 多層防御の一翼 (B 層)

---

## BE-02: ProfileComponent

### `POST /profile` (create/update profile after hearing = ダメな欲望プロファイル登録)
- **Input**:
  ```yaml
  UserProfile:  # ダメな欲望プロファイル
    punctualityTime: string (HH:mm, default "18:00")  # ダメモード突入時刻
    area: string (district level, e.g. "品川区")
    trainLines: string[]  # D6
    hobbyCategories: string[] (enum: sauna|gym|movie|anime|oshi|game|other)  # D5
    familyMembers:  # D1
      - type: enum (spouse|child|pet)
        nickname: string
    petInfo:  # D1
      - type: enum (cat|dog|other)
        nickname: string
    foodPreferences: string[] (enum: izakaya|ramen|cafe|other)  # D3
    drinkPreferences:  # D4
      genres: string[] (enum: beer|sour|highball|sake|wine|none)
    oshi:  # D2
      - name: string (free text, e.g. 推しの名前・配信者名)
    gameGenres: string[]  # D5
    damnIntensity: integer (1-5, ダメモード強度 = 煽り強度)
  ```
- **Output**: `UserProfile` (保存後の正規化済)
- **Errors**: `ValidationError`, `AuthenticationError`
- **Side Effects**: DynamoDB `USER#<userId>/PROFILE` upsert, AuditComponent.record (EventBridge)
- **Idempotence**: 同一ペイロードの再送で状態不変 (PBT-04)

### `GET /profile` → `UserProfile`
- **Errors**: `NotFoundError`, `AuthenticationError`
- **Side Effects**: read-only

### `PATCH /profile` (partial update)
- **Input**: UserProfile の部分セット (JSON Merge Patch)
- **Output**: 更新後の `UserProfile`
- **Side Effects**: 同上、AuditComponent.record (before/after 差分)

### `DELETE /profile` (アカウント削除の一部)
- **Side Effects**: プロファイル削除 + 関連リソース削除ジョブ起動 (EventBridge `account.deletion.requested`)
- **Errors**: `AuthenticationError`

---

## BE-03: SchedulerComponent (Reference Implementation, = 堕落ランプ reference 実装)

### `next_present_at(session_start: ISO8601, now: ISO8601, punctuality: HH:mm) → ISO8601`
- **Purpose**: PBT-05 oracle。クライアントのスケジューラ (= 堕落ランプ) と同一ロジックを "reference table" で実装
- **Input**: セッション開始時刻、現在時刻、ユーザー定時 (= ダメモード突入時刻)
- **Output**: 次回提示時刻 (UTC ISO8601)
- **Errors**: `ValidationError`
- **Notes**: 純粋関数、副作用なし。Hypothesis / fast-check で property テスト

### `compute_rate_table(remaining_minutes: integer) → integer` (interval in minutes, = 堕落ランプの核)
- **Purpose**: レート計算の基礎関数 (table-driven) — 堕落ランプ単調非増加の定義
- **Rate table** (FR-02-01 準拠、堕落ランプ):
  ```
  remaining >= 40 minutes → 12 minutes interval  (仕事モードに軽くヒビ)
  20 <= remaining < 40   → 7 minutes interval   (ダメな欲望が視界を侵食)
  5  <= remaining < 20   → 3 minutes interval   (仕事モード崩壊寸前)
  0  <= remaining < 5    → 2 minutes interval   (ダメモード寸前)
  remaining < 0          → stop (return None/error sentinel、堕落ゲート発動へ)
  ```
- **Output**: 次までの待機分数 (integer)
- **Invariants** (PBT-03):
  - input が減少すると output も単調非増加 (堕落ランプ不変条件)
  - output は常に正の整数

### `POST /sessions/start` / `POST /sessions/stop`
- **Purpose**: クライアントスケジューラの開始/停止を記録 (履歴可視化用)
- **Input**: `{}` (userId は JWT から)
- **Output**: `{ sessionId: string, startedAt / stoppedAt }`

---

## BE-04: ContentSelectionComponent (= ダメな未来ジェネレータ)

### `POST /content/next`
- **Purpose**: クライアントスケジューラ (堕落ランプ) が提示時刻になったらコールする主 API — "ダメな未来" を具象化して返す
- **Input**:
  ```yaml
  ContentRequest:
    context:
      timestamp: ISO8601
      location: { lat: number, lng: number, district: string }  # district = 区レベル
      weather: { condition: enum(sunny|cloudy|rain), temperature: number }  # dummy でも OK
      trainStatus: { line: string, delayMinutes: integer }  # dummy
  ```
- **Output**:
  ```yaml
  ContentResponse:
    contentId: string
    category: enum (restaurant|hobby|train|weather|family|pet|drink|other)  # drink = D4 酒
    desireCategory: enum (D1|D2|D3|D4|D5|D6|D7)  # ダメ欲望カテゴリ
    title: string
    body: string  # AI-generated 堕落煽りコピー, 20-80 chars
    imageUrl?: string
    actions: ["SEEN", "IGNORE", "LEAVE_NOW"]  # LEAVE_NOW = もう堕ちる
  ```
- **Errors**: `UpstreamError` (Bedrock 失敗時 → フォールバック使用), `ValidationError`
- **Side Effects**:
  1. Profile (ダメな欲望プロファイル) を取得
  2. Recent history を取得 (直近 N=10)
  3. Bedrock (Claude) をコール (5 秒 timeout)
  4. 応答パース → sanitize → HistoryComponent.append
  5. 失敗時: ContentRepository.random_by_category() + template-based copy (堕落煽り途切れさせない)
  6. D4 酒で「飲まない」ユーザーはプロンプトから D4 を除外
- **Security Rules**: SEC-03 (PII のログ除外), SEC-05 (プロンプト長上限), SEC-15 (fail-safe)

### `generate_caption(photo_metadata, user_profile, context) → string` (internal)
- **Purpose**: US-04-02, US-03-04 用の堕落煽り文言生成ヘルパ (公開 API ではなく ContentSelection 内部で呼ばれる)
- **Output invariant**: 20-60 文字以内 (PBT-03)

---

## BE-05: ContentRepositoryComponent

### `find_restaurants_by_district(district: string, category: enum, limit: integer=3) → Restaurant[]`  # D3
### `find_by_hobby_category(hobby: enum, limit: integer=5) → HobbyContent[]`  # D5
### `find_train_status(line: string) → TrainInfo`  # D6
### `find_weather(district: string) → WeatherInfo`  # D6
### `find_drinks_by_genre(genre: enum, limit: integer=5) → DrinkContent[]`  # D4 酒テロダミーデータ
- All read-only, dummy data seeded at deploy time (D3 20+件, D4 25+件, D5 5件/sub-cat, D6 dummy)
- **Errors**: `NotFoundError`

---

## BE-06: PhotoComponent

### `POST /photos/upload-url`
- **Purpose**: Pre-signed PUT URL 発行
- **Input**:
  ```yaml
  UploadUrlRequest:
    filename: string  # basename only, サーバ側で拡張子と MIME 再検証
    contentType: enum (image/jpeg|image/png|image/webp)
    sizeBytes: integer  # クライアント申告、サーバで上限チェック
    tag: enum (family|pet|other)
  ```
- **Output**:
  ```yaml
  UploadUrlResponse:
    photoId: string (uuid)
    presignedUrl: string (S3 PUT URL, 15min expiry)
    s3Key: string (user/<userId>/photos/<uuid>.<ext>)
  ```
- **Errors**: `ValidationError` (MIME / ext / size 不正), `AuthenticationError`
- **Side Effects**: Photo メタ (status=PENDING) を DynamoDB に保存
- **Security Rules**: SEC-01, SEC-05, SEC-08 (自分のプレフィックスのみ), SEC-09

### `POST /photos/{photoId}/confirm`
- S3 への upload 完了通知、status=ACTIVE に更新
- S3 Object の存在とサイズ検証 (HEAD)
- **Side Effects**: 確認できなければ Pre-signed URL は自然失効

### `GET /photos`
- 自分のアクティブ写真一覧
- **Errors**: `AuthenticationError`

### `DELETE /photos/{photoId}`
- S3 delete + メタ update (soft delete or hard delete)
- **Side Effects**: AuditComponent (EventBridge)
- **Errors**: `AuthorizationError` (IDOR), `NotFoundError`

---

## BE-07: HistoryComponent

### `POST /history` (internal, from ContentSelection)
- **Input**: `{ userId, contentId, category, generatedBody, presentedAt }`
- **Output**: `{ historyId }`
- **Side Effects**: DynamoDB append

### `GET /history?days=7` (external)
- 直近 N 日の履歴取得
- **Errors**: `AuthenticationError`

### `get_recent(user_id, n=10)` (internal helper)
- ContentSelection が直前コンテンツ履歴を取得するのに使う

---

## BE-08: ReactionComponent

### `POST /reactions`
- **Input**:
  ```yaml
  ReactionRequest:
    contentId: string
    action: enum (SEEN|IGNORE|LEAVE_NOW)
    clientIdempotencyKey: string (uuid)  # 冪等キー
    timestamp: ISO8601
  ```
- **Output**: `{ reactionId }`
- **Errors**: `ValidationError`, `AuthenticationError`, `AuthorizationError` (IDOR: contentId が自分の履歴に無い)
- **Side Effects**:
  1. DynamoDB put-if-absent (冪等キーで重複排除)
  2. `action=LEAVE_NOW` → EventBridge `reaction.leave_now` 発火 (TerminationComponent へ、D1=D)
- **PBT Rules**: PBT-04 (同一冪等キーは必ず同一結果)

---

## BE-09: TerminationComponent (= 堕落ゲート発動記録)

### `POST /terminations/clock-out`
- **Purpose**: 堕落ゲートのダミー勤怠宣言ボタンが押されたときに呼ばれる
- **Input**:
  ```yaml
  TerminationRequest:
    trigger: enum (PUNCTUALITY|EARLY)  # 定時発火 (堕落ゲート自動発動) vs 早期堕ち
    timestamp: ISO8601
  ```
- **Output**:
  ```yaml
  TerminationResponse:
    recorded: boolean  # 同日2度目以降は false (堕落ゲート日次冪等)
    message: string  # AI-generated ダメモード突入メッセージ (20-50 chars)
  ```
- **Errors**: `AuthenticationError`
- **Side Effects**:
  1. DynamoDB `USER#<userId>/TERMINATION#<date>` put-if-absent (堕落ゲート日次一意)
  2. 2 度目以降は recorded=false で早期 return
  3. ContentSelection.generate_caption() 相当のプロンプトでダメモード突入メッセージを生成
  4. AuditComponent (EventBridge)
- **PBT Rules**: PBT-04 (日次冪等)

### `handle_leave_now_event(event)` (EventBridge consumer)
- ReactionComponent からの `LEAVE_NOW` (もう堕ちる) イベントを受けて、早期堕ちのサーバー側記録を先行作成
- (クライアントが堕落宣言ボタンを押すまで recorded=false、押されたら true に更新)

---

## BE-10: NotificationComponent (Full, 最小実装)

### `POST /notifications/subscriptions` / `DELETE /notifications/subscriptions/{id}`
- Web Push 購読の登録/解除 (VAPID 公開鍵を返却、購読情報を DynamoDB に保存)
- MVP ではエンドポイントだけ用意、actual push は Full で実装

---

## BE-11: AuditComponent

### `record_event(event: AuditEvent)` (EventBridge consumer)
- **Input**:
  ```yaml
  AuditEvent:
    actor: string (userId)
    action: enum (PROFILE_UPDATE|PHOTO_DELETE|ACCOUNT_DELETE|TERMINATION|LEAVE_NOW|...)
    resourceType: string
    resourceId: string
    before: object
    after: object
    timestamp: ISO8601
    sourceIp: string
  ```
- **Side Effects**: append-only DynamoDB table (IAM で delete 不可) + CloudWatch structured log (2 系統冗長)
- **Security Rules**: SEC-03, SEC-13, SEC-14

---

## BE-12: InfrastructureComponent (IaC, CDK)

メソッドではなくスタック定義。主要スタック:

- `NetworkStack` — (MVP は VPC 不要、CloudFront + API Gateway 直下)
- `AuthStack` — Cognito User Pool / App Client / Pre-signup Lambda トリガー
- `DataStack` — DynamoDB (AfterFiveTable, AuditTable), S3 (PhotoBucket: Public Access Block + OAC), KMS CMK
- `ApiStack` — API Gateway REST, Lambda functions (per-route), Lambda Authorizer, Usage Plan (rate limiting)
- `FrontendStack` — CloudFront + S3 (Web PWA 配信), Response Headers Policy (CSP / HSTS / 他), OAC
- `ObservabilityStack` — CloudWatch Dashboards, Alarms (auth failure rate, 5xx, Bedrock error), Log Insights queries
- `AiStack` — Bedrock access (IAM only, Bedrock 自体は AWS マネージド)

---

# Frontend Component Methods

## FE-01: AppShell

### `initialize() → Promise<void>`
- PlatformAdapter / ApiClient 初期化、認証状態復元

### `route(path: string)`
- Router API (React Router)

---

## FE-02: OnboardingModule

### `signUp(email: string, password: string) → Promise<User>`
- Cognito Hosted SDK (Amplify Auth or amazon-cognito-identity-js) 経由

### `login(email: string, password: string) → Promise<Session>`
### `logout() → Promise<void>`
### `submitHearing(answers: HearingAnswers) → Promise<UserProfile>`
### `updatePunctuality(time: HHmm) → Promise<UserProfile>`
### `editProfile(patch: Partial<UserProfile>) → Promise<UserProfile>`
- すべて ApiClient 経由で BE-02 を呼ぶ

---

## FE-03: ReminderFeedModule

### `onContentReceived(content: ContentResponse) → void`
- フォアグラウンドならモーダル/トースト、バックグラウンドなら NotificationAdapter.notify

### `onReaction(action: ReactionAction) → Promise<void>`
- ApiClient 経由で BE-08 に POST
- `action=LEAVE_NOW` なら TerminationOverlayModule.show() を即座に呼ぶ

---

## FE-04: NotificationAdapter (Interface)

```typescript
interface NotificationAdapter {
  requestPermission(): Promise<PermissionState>
  notify(title: string, body: string, options?: NotifyOptions): Promise<void>
  isAvailable(): boolean
}
```

- `TauriNotificationAdapter`: `tauri-plugin-notification` 経由
- `WebNotificationAdapter`: `window.Notification`

---

## FE-05: TerminationOverlayModule (= 堕落ゲート UI)

### `show(trigger: 'PUNCTUALITY'|'EARLY') → Promise<void>`
- 堕落ゲート (全画面オーバーレイ) + SE + アニメーション + 「今日の仕事は終わりだ、明日の自分に任せてダメになれ」コピー表示
- ESC 無効、堕落宣言ボタン押下まで閉じない

### `onClockOut() → Promise<void>`
- ApiClient → BE-09 `POST /terminations/clock-out`
- 返ってきた message (= ダメモード突入メッセージ) を表示

---

## FE-06: PhotoManagerModule

### `uploadPhoto(file: File, tag: PhotoTag) → Promise<Photo>`
1. ApiClient → BE-06 `POST /photos/upload-url` で presigned URL 取得
2. `fetch(presignedUrl, { method: 'PUT', body: file })`
3. ApiClient → BE-06 `POST /photos/{id}/confirm`

### `listPhotos() → Promise<Photo[]>`
### `deletePhoto(photoId: string) → Promise<void>`

---

## FE-07: SchedulerClient (= 堕落ランプ)

### `startSession(punctuality: HHmm) → void`
- `setInterval` で 1 分ごとに `compute_rate_table(remaining)` を評価 (= 堕落ランプを刻む)
- 提示時刻になれば `onTrigger()` を発火 (呼び出し側: ReminderFeedModule)

### `stopSession() → void`
### `onTrigger(callback: () => Promise<void>) → void` (event emitter API)
### `nextPresentAt(): Date` (テスタビリティのため公開、PBT-05 の oracle = BE-03 と比較される)

**内部 invariant (PBT-03)**:
- `stopSession()` 後は `onTrigger` が発火しない
- `remaining` が同じなら `nextPresentAt()` の結果も同じ (純粋関数)
- `startSession` は冪等 (既に開始中なら何もしない) (PBT-04)
- 堕落ランプ単調非増加: remaining が減少 → interval も単調非増加

---

## FE-08: ApiClient

### `request<T>(config: RequestConfig): Promise<T>`
- JWT 自動添付 (Authorization: Bearer)
- 401 時は 1 回だけ refreshToken 試行、失敗なら logout
- タイムアウト / リトライ (べき等メソッドのみ)

### `get / post / patch / delete` ショートハンド

- 型: OpenAPI から生成した TypeScript 型で保護

---

## FE-09: PlatformAdapter (Interface)

```typescript
interface PlatformAdapter {
  getCurrentLocation(): Promise<Coordinates | null>  // 区レベルに丸める
  bringWindowToFront(): Promise<void>  // 定時オーバーレイ用
  isAppInForeground(): boolean
  playSound(url: string): Promise<void>  // SE
  readFileAsBlob(path: string): Promise<Blob>  // Tauri only
}
```

- `TauriPlatformAdapter`: Rust command 経由 (`@tauri-apps/api`)
- `WebPlatformAdapter`: `navigator.geolocation`, `document.visibilityState`, `new Audio()`

---

# Cross-Cutting Method Overview

## X-01: Observability (Logger)

```python
# Lambda Powertools Logger
from aws_lambda_powertools import Logger, Tracer, Metrics

logger = Logger(service="afterfive", sampling_rate=1.0)
tracer = Tracer(service="afterfive")
metrics = Metrics(namespace="AfterFive", service="afterfive")

@logger.inject_lambda_context
@tracer.capture_lambda_handler
@metrics.log_metrics
def handler(event, context):
    ...
```

- PII redaction middleware を `logger.append_keys` ではなく `format` フィルタで注入

## X-02: SharedModels

- `openapi/schema.yaml` がソース
- バックエンド: `datamodel-code-generator` or `openapi-python-client` で Pydantic モデル生成
- フロント: `openapi-typescript` で TS 型生成
- `npm run gen:types` / `poetry run gen:models` で再生成可能

## X-03: RepositoryLayer

```python
class ProfileRepository:
    def __init__(self, table): self.table = table
    def get(self, user_id: str) -> UserProfile | None: ...
    def put(self, user_id: str, profile: UserProfile) -> None: ...  # idempotent (PBT-04)
    def patch(self, user_id: str, patch: dict) -> UserProfile: ...
    def delete(self, user_id: str) -> None: ...
```

類似の Repository が Photo / History / Reaction / Termination / Content / Audit 用に存在。全てが Single-Table + Composite PK (partitionKey=USER#<id>, sortKey 種別別) にアクセス。

## X-04: ErrorHandling

```python
class AppError(Exception):
    status_code: int = 500
    error_code: str = "INTERNAL_ERROR"

class ValidationError(AppError):
    status_code = 400
    error_code = "VALIDATION_ERROR"

class AuthenticationError(AppError): status_code = 401; error_code = "AUTH_FAIL"
class AuthorizationError(AppError):  status_code = 403; error_code = "FORBIDDEN"
class NotFoundError(AppError):       status_code = 404; error_code = "NOT_FOUND"
class ConflictError(AppError):       status_code = 409; error_code = "CONFLICT"
class UpstreamError(AppError):       status_code = 502; error_code = "UPSTREAM_FAIL"

def error_response(e: Exception) -> dict:
    if isinstance(e, AppError):
        return {"statusCode": e.status_code, "body": {"errorCode": e.error_code, "message": str(e)}}
    # 未捕捉例外は logger.exception に加えて汎用 500
    return {"statusCode": 500, "body": {"errorCode": "INTERNAL_ERROR", "message": "Internal server error"}}
```

---

# NFR Coverage Summary (Method Level)

## Security Baseline

| SEC Rule | Where Enforced (method/component) |
|---|---|
| SEC-01 暗号化 | BE-12 InfrastructureComponent: DynamoDB / S3 SSE-KMS |
| SEC-02 NW ログ | BE-12: CloudFront/API Gateway アクセスログ |
| SEC-03 アプリログ | X-01 Observability + PII filter |
| SEC-04 HTTP Headers | BE-12 FrontendStack (CloudFront Response Headers Policy) |
| SEC-05 入力検証 | X-02 SharedModels (Pydantic) + 全 POST/PATCH handler |
| SEC-06 最小権限 | BE-12: Lambda ロール細分化 |
| SEC-07 NW 制限 | BE-12: CloudFront + API Gateway のみ公開 |
| SEC-08 認可 | E2=D 多層防御: A Authorizer, B BE-01 @requires_ownership, C X-03 Repository の userId where |
| SEC-09 ハードニング | X-04 ErrorHandling (汎用メッセージ), BE-12: S3 Public Block |
| SEC-10 Supply Chain | CI (後段 Build and Test ステージ): `pip-audit`, `npm audit`, lockfile pinning |
| SEC-11 安全な設計 | 認可は AuthComponent 集中、レート制限は BE-12 Usage Plan |
| SEC-12 認証 | BE-01 + BE-12 AuthStack (パスワードポリシー) |
| SEC-13 完全性 | BE-11 AuditComponent (audit table), SRI (FE-05 SE 素材) |
| SEC-14 監視 | BE-12 ObservabilityStack: alarms + ≥90日保持 |
| SEC-15 例外処理 | X-04 ErrorHandling + 各 handler の top-level try/except |

## Property-Based Testing

| PBT Rule | Where Enforced |
|---|---|
| PBT-01 Property Identification | Per-unit Functional Design で各コンポーネントを精査 |
| PBT-02 Round-Trip | X-02 SharedModels の serialize/deserialize、X-03 Repository put/get |
| PBT-03 Invariant | BE-03 `compute_rate_table`、BE-04 文字数制約、FE-07 SchedulerClient |
| PBT-04 Idempotence | BE-02 profile put、BE-06 photo confirm、BE-08 reaction、BE-09 termination、FE-07 `startSession` |
| PBT-05 Oracle | BE-03 reference table を FE-07 SchedulerClient のテストで比較 |
| PBT-06 Stateful | FE-07 SchedulerClient + 状態モデル / BE-07+BE-08 の一連コマンド |
| PBT-07 Generator Quality | ヒアリング / 位置 / 日本語生成器を X-02 with custom strategies で提供 |
| PBT-08 Shrinking | Hypothesis / fast-check のデフォルト有効、CI でシード記録 |
| PBT-09 Framework | Python=Hypothesis / TypeScript=fast-check (NFR-05 で既定) |
| PBT-10 Complementary | 各 Story AC を example-based E2E としても実装 (Build and Test) |
