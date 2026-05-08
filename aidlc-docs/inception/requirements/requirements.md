# Requirements — アフターファイブ

**Version**: 1.1
**Last Updated**: 2026-05-08
**Project Type**: Greenfield / 「人をダメにするサービス」プロダクト開発

---

## 1. Intent Analysis Summary

| 項目 | 内容 |
|---|---|
| **User Request** | 「人をダメにするサービス」というテーマで、定時退社を仕掛けるアプリ「アフターファイブ」を AI-DLC を使って構築したい |
| **Request Type** | New Project (Greenfield) |
| **Scope Estimate** | System-wide (フロントエンド / バックエンド / AI / データストア / 認証 / インフラまで含む) |
| **Complexity Estimate** | Complex (AI 連携、マルチチャネル配信 <Desktop + Web>、外部コンテキスト連携、動的スケジューリング) |
| **Primary Goal** | 「明日の自分に任せて今日は精一杯楽しむ」というコンセプトを体現する、定時退社を強く後押しするプロダクトを実装すること |
| **Product Focus** | 働き手の退勤動線を設計で支援し、家族・推し・趣味・食・天気などの「別の未来」をユーモラスな AI コピーで提示する |

### 1.1 Concept

- **テーマ**: 人をダメにするサービス
- **コンセプト**: 明日の自分に任せて今日は精一杯楽しむ
- **プロダクトビジョン**: 定時 1 時間前から、仕事とは無関係な魅力的な「別の未来」を働き手の視界に差し込み、定時に「勤怠入力して帰る」動線まで一気に導く。
- **UI トーン**: ユーモラスで煽り系 (「今帰らないと雨に濡れるぞ」「推しが配信始まったぞ」) — ただし AI 生成で動的に文言を作る。

---

## 2. Functional Requirements

### FR-01: ユーザー認証・オンボーディング

#### FR-01-01: ユーザー登録・ログイン
- **MUST** Amazon Cognito User Pool によるメール + パスワード認証を提供すること。
- **MUST** セッションは httpOnly / Secure / SameSite を適切に設定し、サーバーサイドで検証すること (SECURITY-08, SECURITY-12)。
- **MUST** ログアウト時に即座にセッション無効化を行うこと。
- **SHOULD** ログインに複数回失敗した場合、指数バックオフまたはアカウントロックアウトで保護すること (SECURITY-12)。

#### FR-01-02: 初回ヒアリング (オンボーディング)
- **MUST** 初回ログイン直後に最低 10 問程度の詳細ヒアリングを行うこと。収集項目は以下を含む:
  - 基本: 定時 (デフォルト 18:00、曜日別ではなく単一設定)、最寄り地域・使用路線
  - 家族・ペット: 有無、呼び方 (AI 文言生成用)
  - 趣味: サウナ / ジム / 映画 / アニメ / 推し / ゲーム(ジャンル)
  - 食の好み: 居酒屋・ラーメン・カフェ等のカテゴリ選好
  - UI トーン希望 (煽り強度)
- **MUST** ヒアリング結果は DynamoDB のユーザープロファイルに永続化すること。
- **MUST** ヒアリング結果は AI 呼び出し時のプロンプトにも毎回含めて、コンテンツ選定・文言生成に利用すること。
- **MUST** 設定画面から後でヒアリング結果を編集できること。

#### FR-01-03: 定時設定
- **MUST** ユーザーは定時をいつでも設定画面で変更できること。デフォルトは `18:00`。
- **SHOULD** タイムゾーンはクライアントの現地時間で動作し、内部は JST 固定でも可 (日本向けのみ)。

---

### FR-02: コア機能 ― 「別の未来」コンテンツ提示

#### FR-02-01: 動的頻度スケジューリング
- **MUST** 定時 1 時間前 (例 17:00) からコンテンツ提示を開始すること。
- **MUST** 提示頻度は定時に近づくほど高くなること (動的スケジューリング)。参考レート:
  - 17:00 〜 17:20: 10〜15 分ごとに 1 件
  - 17:20 〜 17:40: 5〜10 分ごとに 1 件
  - 17:40 〜 18:00: 2〜3 分ごとに 1 件 (終盤はほぼ連続的)
- **MUST** 定時 (18:00) 時点で提示を止め、終了動線 (FR-04) へ遷移すること。
- **MUST** コンテンツ選定は Amazon Bedrock (Claude 系モデル) にコンテキスト (現在時刻・位置・天気・ユーザープロファイル・最近提示したコンテンツ履歴) を渡して決定すること。

#### FR-02-02: コンテンツカテゴリ
各コンテンツカテゴリは独立モジュールとして、MVP では以下の範囲で実装する。

| カテゴリ | MVP 実装範囲 |
|---|---|
| **家族・ペット写真** | 画像アップロード (S3 Pre-signed URL) + AI で呼び名・時刻・天気を踏まえたキャプション生成 (例「今日も◯◯が待ってるよ」) |
| **近隣の飲食店** | 位置情報 + **ダミーデータ** (JSON ファイルまたは DynamoDB にシード) |
| **趣味系 (サウナ/ジム/映画/アニメ/推し/ゲーム)** | 事前仕込みの **ダミーデータ** のみ (雰囲気重視) |
| **電車の混雑・遅延** | **ダミーデータ** による演出 |
| **天気** | **固定/ダミーデータ** (「まもなく雨が降る」等を時刻で切替) |

- **MUST** 各コンテンツは「カテゴリ・タイトル・本文 (AI 生成)・画像 URL・提示時刻」を持つ統一データ構造で扱うこと。
- **MUST** AI 生成の文言は **ユーモラス・煽り系** のトーンに統一すること (プロンプトで指示)。
- **MUST** 同一カテゴリが短時間に偏らないよう、提示履歴を考慮して多様性を確保すること。

#### FR-02-03: 位置情報取得
- **MUST** ブラウザ / Tauri の Geolocation API で現在位置を取得すること。
- **MUST** ユーザーが許可しなかった場合はデフォルト地域 (例: プロファイルの最寄り地域) を使用するフォールバックを持つこと。
- **MUST** 位置情報は区レベル粒度までの丸め込みを行ってから AI プロンプト・ログに使用すること (SECURITY-03: PII を最小化)。

#### FR-02-04: コンテンツ提示 UI
- **MUST** コンテンツはポップアップ / カード形式で目立つように画面手前に表示されること。
- **MUST** アプリ起動中はアプリ内通知 (モーダル / トースト)、バックグラウンド時は OS ネイティブ通知 (Tauri 版は `tauri-plugin-notification`、Web 版は Web Notifications API) を使うこと。
- **SHOULD** ユーザーは「見た」「無視した」等の軽量なアクションができること (FR-03 で履歴保存)。

---

### FR-03: 履歴・リアクション

#### FR-03-01: 提示履歴の永続化
- **MUST** 各ユーザーへのコンテンツ提示を DynamoDB に保存すること (userId, timestamp, category, contentId, generatedText)。
- **MUST** 履歴は AI 呼び出し時のプロンプトに直近 N 件を含めて、重複を避けること。

#### FR-03-02: リアクションの記録
- **MUST** ユーザーのリアクション (見た / 無視した / 帰ったボタンを押した) を DynamoDB に保存すること。
- **SHOULD** 将来的な学習データとして蓄積可能な構造にすること (MVP 内で学習は行わない)。

---

### FR-04: 終了動線 (勤怠入力画面の提示)

#### FR-04-01: ダミー勤怠画面の提示
- **MUST** 定時 (設定された時刻) になった瞬間、**全画面オーバーレイ** で擬似勤怠画面を表示すること。
- **MUST** オーバーレイは SE (効果音) + アニメーション (スライドイン / フラッシュ等) を伴い「帰るぞ!」と強く促すこと。
- **MUST** アプリ内のダミー勤怠画面を実装 (退勤ボタン 1 個 + 「お疲れさまでした!」演出)。API 連携は不要。
- **MUST** 退勤ボタンは **ユーザー本人が押す** ことを前提にする (自動クリックは行わない)。
- **MUST** ボタン押下後、退勤完了画面 (「今日もおつかれ。明日の自分に任せよう」等の一言) を表示してコンテンツ配信を終了すること。

---

### FR-05: 画像アップロード

#### FR-05-01: 家族・ペット写真のアップロード
- **MUST** フロントからの画像アップロードは S3 Pre-signed URL 方式で直接 S3 に PUT すること (API Gateway 経由しない)。
- **MUST** Pre-signed URL 発行 API は認証済みユーザーのみが利用でき、URL はそのユーザー専用のプレフィックス (`user/{userId}/photos/`) に限定すること (SECURITY-08 IDOR 対策)。
- **MUST** 画像のメタ情報 (userId, s3Key, uploadedAt, caption) は DynamoDB に保存すること。
- **MUST** S3 バケットはパブリックアクセスをブロックし、配信は CloudFront 経由で署名付き URL または OAC (Origin Access Control) 経由で行うこと (SECURITY-01, SECURITY-09)。
- **MUST** ファイルサイズ上限 (例 10MB)、許可拡張子 (jpg/png/webp)、MIME タイプ検証を実施すること (SECURITY-05)。

---

### FR-06: マルチチャネル配信 (Desktop + Web)

#### FR-06-01: 同一コードベースで 2 ビルド
- **MUST** フロントエンドは同じ React + TypeScript + Vite のソースコードから:
  - **Tauri ビルド**: macOS / Windows 向けデスクトップアプリ
  - **Web ビルド**: CloudFront + S3 ホスティングの PWA
  の 2 系統を生成できること。
- **MUST** プラットフォーム差分 (通知 API、ファイル保存場所、Geolocation 実装差) はアダプタ層 (ポート & アダプタ) で吸収すること。
- **MUST** 両方とも同一のバックエンド API (API Gateway) に接続すること。

---

## 3. Non-Functional Requirements

### NFR-01: パフォーマンス
- コンテンツ提示のエンドツーエンドレイテンシ (トリガー → 表示) は 5 秒以内を目標とする。
- Bedrock 呼び出しはタイムアウト 5 秒、フォールバックとして事前生成テンプレート文言を持つ。

### NFR-02: スケーラビリティ
- サーバーレス構成 (Lambda + DynamoDB + API Gateway) により水平スケール。
- 初期想定負荷は同時 100 ユーザー程度 (実計測は行わない)。

### NFR-03: セキュリティ (Security Baseline Extension — **Enforced**)

| SECURITY Rule | 本プロジェクトへの適用方針 |
|---|---|
| SECURITY-01: Encryption at Rest/Transit | DynamoDB 暗号化、S3 SSE、TLS 1.2+ 強制、CloudFront HTTPS-only |
| SECURITY-02: Access Logging on Network Intermediaries | CloudFront / API Gateway アクセスログを CloudWatch Logs へ |
| SECURITY-03: Application-Level Logging | Lambda Powertools for Python による構造化ログ (correlation id 付き)、PII・認証トークンは除外 |
| SECURITY-04: HTTP Security Headers | CloudFront Response Headers Policy で CSP / HSTS / X-Content-Type-Options / X-Frame-Options / Referrer-Policy を付与 |
| SECURITY-05: Input Validation | API 入力は Pydantic でスキーマ検証、S3 アップロードは MIME / サイズ / 拡張子チェック |
| SECURITY-06: Least-Privilege IAM | Lambda ロールは必要最小限の資源を明示、ワイルドカード禁止 |
| SECURITY-07: Restrictive Network Configuration | Lambda はデフォルト VPC 外 (パブリックサービスのみ)、バケット・DynamoDB は Public Access Block |
| SECURITY-08: Application-Level Access Control | Cognito JWT をサーバー側で毎回検証、オブジェクト所有権チェック (userId == token.sub)、CORS は明示オリジン (Web 版) + Tauri スキーム |
| SECURITY-09: Security Hardening | デフォルト認証情報禁止、エラーは汎用メッセージ、S3 Public Access Block |
| SECURITY-10: Supply Chain | ロックファイル (package-lock.json, poetry.lock 等) コミット、`npm audit` / `pip-audit` を CI で実行、Docker 未使用だが Lambda レイヤーはピン留め |
| SECURITY-11: Secure Design | 認可/認証は専用モジュール、API に Rate Limiting (API Gateway Usage Plan)、ミスユースケース 1 件以上を設計ドキュメントに記載 |
| SECURITY-12: Authentication | Cognito のパスワードポリシー (8 文字以上、大文字小文字数字記号) + Advanced Security Features (推奨)、MFA は管理者用 (MVP では Optional) |
| SECURITY-13: Integrity | 安全なデシリアライズ、Web 版は SRI hash を外部スクリプトに付与、クリティカルなデータ変更 (プロファイル更新) は監査ログ |
| SECURITY-14: Alerting & Monitoring | CloudWatch Alarms: 認証失敗レート、5xx レート、Log Retention 最低 90 日 |
| SECURITY-15: Exception Handling | fail-closed 原則、Lambda ハンドラのトップレベルで catch、エラーレスポンスは一般化 |

### NFR-04: 信頼性・可用性
- MVP はベストエフォート。外部依存 (Bedrock) の失敗時は事前生成テンプレートにフォールバック。
- DynamoDB は PAY_PER_REQUEST モードで可用性を確保。

### NFR-05: テスト (Property-Based Testing Extension — **Enforced**, Full mode)

| PBT Rule | 本プロジェクトへの適用方針 |
|---|---|
| PBT-01: Property Identification | Functional Design 段階で全 Unit について testable property を洗い出す |
| PBT-02: Round-Trip | プロファイル / ヒアリング結果 / 履歴の JSON シリアライズ・DynamoDB アイテム変換の往復 |
| PBT-03: Invariants | スケジューラの頻度計算 (残り時間が減るほど頻度 ≥ 前のレート)、コンテンツの重複排除ルール |
| PBT-04: Idempotence | プロファイル更新、Pre-signed URL 発行、リアクション記録 (冪等 key 設計) |
| PBT-05: Oracle | スケジューラは単純な定数テーブル (time → rate) を reference model として比較 |
| PBT-06: Stateful PBT | コンテンツ提示履歴 + リアクションを保持する状態モデルへの一連コマンド (present / react) を生成し不変条件 (履歴連続性・ユーザー分離) を検証 |
| PBT-07: Generator Quality | 日本時刻・日本語文字列・位置情報 (東京 23 区近辺)・ユーザープロファイルのドメイン生成器を定義 |
| PBT-08: Shrinking / Reproducibility | Hypothesis (Python) と fast-check (TS) の shrinking を有効化、CI でシード記録 |
| PBT-09: Framework Selection | **Python: Hypothesis**、**TypeScript: fast-check** |
| PBT-10: Complementary | 重要ビジネスパス (定時オーバーレイ発動、ボタン押下後の遷移) は example-based もセットで実装 |

### NFR-06: 言語・ロケール
- UI・AI 生成文言は **日本語のみ**。
- タイムゾーン: Asia/Tokyo (JST, UTC+9) 固定。

### NFR-07: 運用・監視
- CloudWatch Dashboard に 主要 KPI (日次アクティブユーザー, Bedrock エラー率, 定時オーバーレイ発動率) をプロット (MVP は最小構成)。

### NFR-08: プライバシー
- 位置情報は区レベルまで粗くしてから保存 / AI プロンプトへ投入。
- 家族・ペット写真は本人用のプレフィックスに隔離、他ユーザーからはアクセス不可。
- Cognito ユーザー削除時に関連データ (プロファイル・画像・履歴) も削除する仕組み (MVP では手動スクリプトで可)。

---

## 4. Key Technical Decisions

| 領域 | 技術選択 | 根拠 |
|---|---|---|
| Frontend | React + TypeScript + Vite | エコシステム豊富、Tauri との相性良好 |
| Desktop Shell | Tauri | ネイティブ軽量、Rust バックエンド部で Geolocation / Notification に OS ネイティブアクセス |
| Web Distribution | CloudFront + S3 (OAC) | サーバーレス、HTTPS・セキュリティヘッダー付与容易 |
| Backend | Python on AWS Lambda | Bedrock SDK (boto3) との親和性、生産性 |
| API | API Gateway REST | 認証統合 (Cognito Authorizer)、Usage Plan でレート制限 |
| Auth | Amazon Cognito User Pool | MVP に十分、JWT 標準対応 |
| Database | Amazon DynamoDB (single-table or 分割) | サーバーレス、暗号化標準、低コスト |
| Object Storage | Amazon S3 (Public Access Block + OAC) | 画像保存・配信 |
| AI | Amazon Bedrock (Claude) | コンテンツ選定・日本語コピー生成 |
| IaC | AWS CDK (Python) or SAM | Python Lambda とのスタック統一。後続フェーズで決定 |
| Test (Python) | pytest + Hypothesis | PBT 対応 |
| Test (TS) | Vitest + fast-check | Vite ネイティブ + PBT 対応 |

---

## 5. Out of Scope (MVP)

- 外部 API 連携 (Google Places / Hot Pepper / OpenWeatherMap / 鉄道遅延 / ゲームセール / 映画 / 推し配信) — すべて **ダミーデータ**
- 実在の勤怠システム連携 — **アプリ内ダミー画面** のみ
- SNS ログイン (Google 等) — メール/パスワードのみ
- 多言語対応 — 日本語のみ
- モバイルネイティブアプリ (iOS/Android) — Web PWA で代替
- AI 学習パイプライン — リアクションを蓄積するだけ、MVP では未使用
- 管理者 (Admin) ダッシュボード

---

## 6. Success Criteria (プロダクト価値)

本プロダクトは「人をダメにするサービス」テーマを真面目に設計で具現化することを目的とする。次を達成基準とする。

| 観点 | 達成基準 |
|---|---|
| **コンセプト体現** | 定時 1 時間前から 18:00 に向けて提示頻度が逓増し、18:00 に全画面オーバーレイ + ダミー勤怠画面で退勤動線を提示する一連の体験が動作する |
| **ユーザー文脈適合** | 初回ヒアリング結果 + 現在時刻 + 位置情報 + 直近履歴を踏まえ、AI がユーモラスな日本語コピーを生成し、カテゴリの偏りを避けて提示する |
| **感情フック** | 家族・ペット写真の AI キャプションと、趣味/食/天気/電車ダミーデータの組み合わせで、4 ペルソナそれぞれの「帰りたい動機」に刺さるコンテンツが最低 1 件発火する |
| **マルチチャネル** | 同一 React コードベースから Tauri デスクトップビルドと Web PWA ビルドの両方が生成でき、どちらも同じバックエンド API で動作する |
| **セキュリティ・プライバシー** | Security Baseline 全 15 ルールと PBT 全 10 ルールが Enforced のまま、blocking finding ゼロで MVP を通す |

---

## 7. Assumptions and Constraints

- ユーザーは日本国内在住で、JST で稼働する。
- Bedrock は東京リージョンまたは最寄り利用可能リージョンでアクセスする (後段のインフラ設計で決定)。
- 開発は本 AI-DLC ワークフローに沿って段階的に成果物を作り上げる。
- MVP と Full は製品フェーズとして分け、MVP = コア体験 (17:00 → 18:00 → 退勤) を通すのに必要な機能、Full = 履歴閲覧・写真管理・アカウント削除など周辺機能、とする。

---

## 8. Traceability — Answers from Requirement Verification

| 要件 | ソース (質問番号) |
|---|---|
| FR-01-01 | Q5 → Clarification Q3 (Cognito User Pool) |
| FR-01-02 | Q15 (C), Q16 → Clarification Q4 (DB + AI) |
| FR-01-03 | Q6 (A: default 18:00, 変更可) |
| FR-02-01 | Q7 (X: 17時低頻度 → 18時高頻度) |
| FR-02-02 | Q8 (Bedrock), Q9 (B), Q10 (B), Q11 (C), Q12 (B), Q13 (B), Q25 (A) |
| FR-02-03 | Q14 (A: Geolocation) |
| FR-02-04 | Q22 → Clarification Q2 (in-app + OS native) |
| FR-03 | Q24 (C) |
| FR-04 | Q17 (B dummy), Q18 (C full-screen overlay) |
| FR-05 | Q23 (A: Pre-signed URL), Q9 (B: upload) |
| FR-06 | Q4 → Clarification Q1 (Tauri + Web 両対応) |
| NFR-03 | Security Baseline opt-in: A (enforce) |
| NFR-05 | Property-Based Testing opt-in: A (enforce) |
| NFR-06 | Q26 (A: Japanese only) |

---

## Appendix A: Glossary

| 用語 | 意味 |
|---|---|
| **別の未来** | 定時 1 時間前からユーザーに提示される、仕事と無関係な魅力的な情報のこと。本アプリのコアコンセプト。 |
| **定時** | ユーザーが設定した退勤目標時刻。デフォルト 18:00。 |
| **終了動線** | 定時到達時にダミー勤怠画面を前面に出し、ユーザー自身に退勤ボタンを押させる演出フロー。 |
| **ヒアリング** | 初回ログイン時にユーザープロファイル (趣味・家族・通勤等) を 10 問以上で収集する処理。 |
| **Tauri** | Rust + Web 技術でデスクトップアプリを作るフレームワーク。 |
