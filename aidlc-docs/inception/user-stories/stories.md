# User Stories — アフターファイブ

**Version**: 1.0
**Last Updated**: 2026-05-07
**Story Format**: Job Story ("When <situation>, I want to <motivation>, so I can <outcome>") — Q1=B
**AC Format**: Given-When-Then (Gherkin) — Q4=A
**Priority Labels**: MVP (2週) / Full (1ヶ月) — Q5=B
**NFR Handling**: 各 Story の AC に Security / PBT / Privacy を織り込む — Q6=C
**Epic Scope**: Q3=X (マルチチャネル配信以外の 5 Epic)

---

## Epic Index

| # | Epic | Goal | MVP/Full 比率 |
|---|---|---|---|
| E1 | オンボーディング | ユーザー登録・プロファイル収集・定時設定 | MVP 寄り |
| E2 | 日中のリマインド体験 | 定時 1 時間前からの動的コンテンツ提示 | MVP 中核 |
| E3 | 退勤動線 | 定時到達時の全画面オーバーレイ + ダミー勤怠 | MVP 中核 |
| E4 | 家族・ペット写真管理 | 写真アップロード + AI キャプション | MVP 要 |
| E5 | プライバシー・セキュリティ | 認証 / データ保護 / 監査の UX | MVP 寄り |

**INVEST 自己レビュー**: 全 Story について I (Independent) / N (Negotiable) / V (Valuable) / E (Estimable) / S (Small, 推定 1-3 days) / T (Testable) を満たすよう設計しました。各 Story の末尾にメモを記載します。

**NFR 織り込みの凡例 (AC 内で明示)**:
- `[SEC-XX]` = Security Baseline ルール (SECURITY-XX)
- `[PBT-XX]` = Property-Based Testing ルール (PBT-XX)
- `[PRV]` = Privacy-related

---

# E1. オンボーディング

> ユーザーに最初の 10 分で「このアプリは自分のためにある」と感じてもらう。

---

## US-01-01: ユーザー登録 [MVP]

**Story**
> **When** 初めてアプリを開いたとき、
> **I want to** メールアドレスとパスワードですぐに登録できるようにしたい、
> **so I can** 面倒な確認なしに体験を開始できる。

**Persona**: P1, P2, P3, P4 (全員)

**Traceability**: FR-01-01, NFR-03 (SECURITY-12)

**AC**
- **Given** 未登録のユーザーがアプリのログイン画面にいる
  **When** メールアドレスとパスワード (8 文字以上・大文字/小文字/数字/記号混合) を入力して「登録」を押す
  **Then** Cognito User Pool にユーザーが作成され、確認メールが送信される `[SEC-12]`
- **Given** 確認メールが届いたユーザー
  **When** メール内のリンク (確認コード入力) を完了させる
  **Then** アプリに自動ログインし、初回ヒアリング画面に遷移する
- **Given** パスワードポリシー違反の入力
  **When** 登録ボタンを押す
  **Then** 汎用的なエラーメッセージ (「パスワードの形式が条件を満たしていません」) が表示され、内部情報は漏洩しない `[SEC-09]`
- **Given** 同一メールで 5 分以内に登録を 3 回失敗
  **When** 4 回目を試みる
  **Then** 一時的にロックされ、指数バックオフ待機が必要になる `[SEC-12]`

**INVEST Notes**: Independent=○ (他 Story 非依存、Cognito のみ)、Small=○ (1 day)

---

## US-01-02: ログイン & セッション維持 [MVP]

**Story**
> **When** 2 回目以降アプリを起動したとき、
> **I want to** 前回のログインを覚えていてほしい、
> **so I can** 毎日のリマインド体験に即座に入れる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-01-01, NFR-03 (SECURITY-08, SECURITY-12)

**AC**
- **Given** 過去 30 日以内にログインしたユーザー
  **When** アプリを起動する
  **Then** JWT リフレッシュトークンが有効ならサーバー側で検証のうえ自動ログインする `[SEC-08]`
- **Given** 有効期限切れトークン
  **When** API をコールする
  **Then** 401 が返り、再ログイン画面に誘導される `[SEC-08]`
- **Given** ログアウトボタンが押された
  **When** サーバーで処理される
  **Then** サーバー側セッションが即時無効化される `[SEC-12]`
- **Given** 一連のトークンシリアライズ処理
  **When** シリアライズ→デシリアライズを往復させる
  **Then** 元のクレームと完全一致する (JSON 往復 property) `[PBT-02]`

**INVEST Notes**: Small=○ (0.5-1 day)

---

## US-01-03: 初回ヒアリング (プロファイル収集) [MVP]

**Story**
> **When** 新規登録直後にオンボーディングに入ったとき、
> **I want to** 10 問程度のヒアリングで自分の生活情報を登録したい、
> **so I can** 自分の趣味・家族・生活圏に合わせた「別の未来」を受け取れる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-01-02, NFR-08 (Privacy)

**AC**
- **Given** 初回ログインを終えた直後のユーザー
  **When** ヒアリング画面に到達する
  **Then** 10 問以上のヒアリング (定時・地域・家族/ペット有無・趣味カテゴリ・食の好み・推し/ゲームジャンル・UI トーン希望 等) が表示される
- **Given** ヒアリングを一問ずつ回答中
  **When** 中断してアプリを閉じる
  **Then** 回答済みの項目は DynamoDB に保存され、再起動時に途中から再開できる
- **Given** 全問回答完了
  **When** 「保存」を押す
  **Then** プロファイルが DynamoDB に確定保存され、「別の未来」提示画面に進む
- **Given** プロファイルドメイン生成器で無作為に生成したプロファイル
  **When** 保存 → 読み出しを往復
  **Then** 完全一致する (`[PBT-02]` round-trip、`[PBT-07]` domain generator)
- **Given** プロファイル保存 API 呼び出し
  **When** 同一 userId で重複コール
  **Then** 結果は冪等 (べき等) で、DB 状態は同一になる `[PBT-04]`
- **Given** 入力フィールド
  **When** 過度に長い文字列や不正 JSON が送信される
  **Then** API Gateway / Lambda がスキーマ検証で 400 を返す `[SEC-05]`

**INVEST Notes**: Small=△ (2-3 days、UI の実装含むが AC で分割可能)

---

## US-01-04: 定時設定 [MVP]

**Story**
> **When** プロファイル画面や設定画面にいるとき、
> **I want to** 自分の定時 (デフォルト 18:00) を設定・変更したい、
> **so I can** 会社独自の就業時間に合わせて退勤リマインドを受けられる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-01-03

**AC**
- **Given** 設定画面を開いたユーザー
  **When** 定時を `19:00` に変更して保存
  **Then** DynamoDB の `punctualityTime` が更新され、スケジューラは翌日から `19:00` 起点で動く
- **Given** 不正な時刻 (例: `25:99`)
  **When** 送信される
  **Then** 400 エラーが返る `[SEC-05]`
- **Given** 任意の有効時刻
  **When** 保存 → 読み出し
  **Then** 同一値 `[PBT-02]`

**INVEST Notes**: Small=○ (0.5 day)

---

## US-01-05: プロファイル編集 [Full]

**Story**
> **When** 数週間使ってプロファイルが実情と合わなくなったとき、
> **I want to** 後からヒアリング回答を編集したい、
> **so I can** コンテンツの精度を保てる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-01-02

**AC**
- **Given** ログイン済みユーザー
  **When** 設定画面から「プロファイル編集」に入る
  **Then** ヒアリング項目が編集可能な形で表示される
- **Given** 編集して保存
  **When** 次回のコンテンツ提示が発生する
  **Then** 新プロファイルが AI プロンプトに反映される
- **Given** 別ユーザーの userId を URL に含めた編集リクエスト
  **When** API がそれを受け取る
  **Then** JWT の sub と比較し 403 を返す (IDOR 対策) `[SEC-08]`

**INVEST Notes**: Small=○ (1 day)

---

# E2. 日中のリマインド体験

> 定時 1 時間前からの動的コンテンツで「早く帰りたい」気持ちを醸成するコア体験。

---

## US-02-01: 動的頻度スケジューラの起動 [MVP]

**Story**
> **When** 自分の定時の 1 時間前になったとき、
> **I want to** 定時が近づくほど頻度が上がるリマインドを自動で受けたい、
> **so I can** 徐々に退勤モードにギアを上げられる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-02-01, NFR-05 (PBT)

**AC**
- **Given** 定時 `18:00` のユーザーが `17:00` を迎える
  **When** スケジューラが起動条件を評価する
  **Then** 提示セッションが開始される
- **Given** 起動したセッション
  **When** 経過時間 `t` (分) が進む
  **Then** 次の提示までの間隔は `t` に対して単調非増加である (17:00 時点で 10-15 分間隔、17:40 時点で 2-3 分間隔) `[PBT-03]` invariant
- **Given** スケジューラの頻度計算ロジック
  **When** 参照実装 (定数テーブル) と比較
  **Then** 全ての経過時間で同一の次回提示時刻を返す `[PBT-05]` oracle
- **Given** 定時 `18:00` に到達
  **When** その瞬間
  **Then** 提示セッションを停止し、退勤動線 (E3) にハンドオフする
- **Given** セッション中の一連のコマンド (start / tick / present / stop) を無作為に生成
  **When** 状態モデルと同期実行
  **Then** 観測可能な状態が両者で一致する `[PBT-06]` stateful PBT

**INVEST Notes**: Small=○ (2 days)

---

## US-02-02: コンテキスト収集 (時刻・位置・天気) [MVP]

**Story**
> **When** コンテンツを選ぶ直前、
> **I want to** 現在の時刻・(許可があれば) 位置情報・天気 (ダミー) を自動で取得したい、
> **so I can** 「今この状況に合った」コンテンツを受け取れる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-02-03, NFR-08 (Privacy)

**AC**
- **Given** ブラウザ / Tauri の Geolocation API
  **When** 許可された
  **Then** 緯度経度を取得し、区レベル (小数 2 桁程度) まで粗くしてから送信する `[PRV][SEC-03]`
- **Given** Geolocation 不許可
  **When** 位置が必要
  **Then** プロファイルの「最寄り地域」にフォールバック
- **Given** 外部依存の位置情報がタイムアウト
  **When** コンテキスト収集が呼ばれる
  **Then** 位置不明として 5 秒以内にデフォルト値で継続 `[SEC-15]` fail-closed の代替 (fail-safe default)

**INVEST Notes**: Small=○ (1 day)

---

## US-02-03: AI によるコンテンツ選定 (Bedrock) [MVP]

**Story**
> **When** スケジューラが提示トリガーを発火したとき、
> **I want to** ユーザープロファイル・履歴・現在コンテキストを踏まえて AI がベストなカテゴリを選んでほしい、
> **so I can** 毎回新鮮で自分に刺さる「別の未来」を受け取れる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-02-01, FR-02-02

**AC**
- **Given** スケジューラが発火
  **When** Lambda がプロファイル + 直近 N 件履歴 + コンテキストをプロンプトに組んで Bedrock (Claude) をコール
  **Then** カテゴリと候補 contentId が返る
- **Given** Bedrock が 5 秒以内に応答しない
  **When** タイムアウト
  **Then** フォールバック (ルールベース + テンプレート文言) でコンテンツを選ぶ `[SEC-15]`
- **Given** 直近 5 件の提示履歴
  **When** 次を選ぶ
  **Then** 同一カテゴリが 3 回連続にならない (多様性 invariant) `[PBT-03]`
- **Given** Bedrock への入力プロンプト
  **When** ログに書き出す
  **Then** PII (メール、本名、正確な座標) は含まれない `[SEC-03]`
- **Given** プロンプトサイズが上限超過
  **When** 入力
  **Then** 事前にトリミングされるか 400 で弾かれる `[SEC-05]`

**INVEST Notes**: Small=△ (2-3 days、プロンプト設計含む)

---

## US-02-04: コンテンツ提示 UI (アプリ内 / OS 通知) [MVP]

**Story**
> **When** アプリからコンテンツを渡されたとき、
> **I want to** アプリ使用中なら画面手前に、非アクティブなら OS 通知で表示したい、
> **so I can** 何をしていても「別の未来」が視界に入る。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-02-04

**AC**
- **Given** アプリがフォアグラウンド
  **When** コンテンツが到着
  **Then** モーダル/トーストで中央に表示され、タイトル・本文・画像が見える
- **Given** アプリがバックグラウンド (Tauri: ウィンドウ非アクティブ、Web: タブ非表示)
  **When** コンテンツが到着
  **Then** Tauri では `tauri-plugin-notification`、Web では Web Notifications API で OS 通知を発火する
- **Given** Web 版で通知権限が拒否されている
  **When** コンテンツが到着
  **Then** フォアグラウンドに戻した際にキューから表示される (失敗黙殺しない) `[SEC-15]`
- **Given** 通知をクリック
  **When** アプリが前面に戻る
  **Then** 該当コンテンツの詳細画面に遷移する
- **Given** コンテンツの表示が HTML レンダリングされる
  **When** AI 生成テキストにスクリプトが混入していても
  **Then** エスケープ/サニタイズされて実行されない `[SEC-05]`

**INVEST Notes**: Small=△ (2 days)

---

## US-02-05: ユーザーリアクション (見た/無視した/帰る) [MVP]

**Story**
> **When** コンテンツを見た後、
> **I want to** 「もう帰る」「あと少し」「興味なし」を軽く選びたい、
> **so I can** 次の提示に反映させられる / いますぐ退勤動線に移行できる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-03-02

**AC**
- **Given** コンテンツ表示中
  **When** 「もう帰る」ボタンを押す
  **Then** リアクション (userId, contentId, action=LEAVE_NOW, timestamp) が DynamoDB に保存され、即座に退勤動線 (E3) に遷移する
- **Given** 「あと少し」
  **When** 押す
  **Then** スケジューラの次回提示を 5 分遅らせる
- **Given** 「興味なし」
  **When** 押す
  **Then** そのカテゴリの次回重み付けが低下する
- **Given** 同一 reaction が重複送信
  **When** API が受ける
  **Then** 冪等 (idempotent) で保存は 1 件のみ `[PBT-04]`
- **Given** 他人の contentId を指定
  **When** リアクション API 呼び出し
  **Then** 403 (IDOR 対策) `[SEC-08]`

**INVEST Notes**: Small=○ (1 day)

---

## US-02-06: コンテンツ: 近隣飲食店 (ダミー) [MVP]

**Story**
> **When** 退勤が近づいてきたとき、
> **I want to** 近所の居酒屋・ラーメン・カフェを画像付きで見たい、
> **so I can** 「ちょっと寄って帰ろう」と思える。

**Persona**: P1, P2, P4

**Traceability**: FR-02-02

**AC**
- **Given** コンテキストに位置情報がある
  **When** 飲食店カテゴリが選ばれる
  **Then** ダミーデータから最寄り 3 件のランダム 1 件を返し、AI が煽り文言を生成する (例「あの二郎が今日は空いてる」)
- **Given** ダミーデータ ロード
  **When** アプリ起動
  **Then** 最低 20 件 (居酒屋/ラーメン/カフェ を均等分布) がシードされている
- **Given** 同一飲食店の連続提示
  **When** 履歴チェック
  **Then** 30 分以内の重複提示はなし `[PBT-03]`

**INVEST Notes**: Small=○ (1 day)

---

## US-02-07: コンテンツ: 趣味系 (サウナ/ジム/映画/アニメ/推し/ゲーム) (ダミー) [MVP]

**Story**
> **When** 自分の趣味カテゴリに合うトリガーが発火したとき、
> **I want to** 事前仕込みのコンテンツをそれっぽく見たい、
> **so I can** デモとして雰囲気が伝わり、実用感を味わえる。

**Persona**: P1 (サウナ/ジム), P2 (アニメ/推し), P4 (ゲーム/アニメ/推し)

**Traceability**: FR-02-02

**AC**
- **Given** ユーザープロファイルに趣味カテゴリ登録あり
  **When** 提示ループでそのカテゴリが選ばれる
  **Then** ダミーデータ (最低 5 件/カテゴリ) からランダム 1 件が AI 煽り文言付きで表示される
- **Given** プロファイルに無いカテゴリ
  **When** 提示ループ
  **Then** 選ばれない (カテゴリ invariant) `[PBT-03]`
- **Given** 全カテゴリで AI 生成された文言
  **When** 文字数/トーン検証
  **Then** 20〜80 文字かつユーモラス (プロンプトで強制) 範囲 (invariant) `[PBT-03]`

**INVEST Notes**: Small=○ (1-2 days)

---

## US-02-08: コンテンツ: 電車・天気 (ダミー) [MVP]

**Story**
> **When** 天気が雨予報だったり電車遅延があったりしたとき (ダミー演出)、
> **I want to** 「今帰らないと巻き込まれる」という警告を受けたい、
> **so I can** 定時退社の "背中を押される"。

**Persona**: P1, P2, P3

**Traceability**: FR-02-02

**AC**
- **Given** 時刻が 17:45
  **When** 天気ダミーデータで「18:15 から雨」がシードされている
  **Then** AI が「あと 30 分で雨。今帰れば濡れない」と生成して提示
- **Given** 使用路線がプロファイルに登録
  **When** 電車遅延ダミーが発火
  **Then** 該当路線の "混雑 / 遅延" 文言を提示
- **Given** 使用路線未登録のユーザー
  **When** 電車カテゴリ
  **Then** 選ばれない `[PBT-03]`

**INVEST Notes**: Small=○ (1 day)

---

## US-02-09: 提示履歴の閲覧 [Full]

**Story**
> **When** 夜に振り返りたいとき、
> **I want to** 今日表示されたコンテンツを一覧で見たい、
> **so I can** どのコンテンツが自分を帰らせたか振り返れる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-03-01

**AC**
- **Given** ログイン済みユーザー
  **When** 履歴画面を開く
  **Then** 直近 7 日分のコンテンツ履歴が時系列で表示される
- **Given** 他人の userId を指定
  **When** 履歴取得 API を叩く
  **Then** 403 `[SEC-08]`

**INVEST Notes**: Small=○ (1 day)

---

# E3. 退勤動線

> 定時になったらユーザー本人を退勤ボタンに導く、本プロダクトの締めくくり。

---

## US-03-01: 定時到達で全画面オーバーレイ起動 [MVP]

**Story**
> **When** 設定した定時 (例 18:00) ぴったりになったとき、
> **I want to** アニメーションと効果音付きで全画面オーバーレイが出てほしい、
> **so I can** 「終業だ」と強く認識できる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-04-01

**AC**
- **Given** 定時 `18:00` を設定したユーザー
  **When** 時刻が 18:00:00 になる
  **Then** 全画面モーダルがスライドインし、効果音 (デフォルト ON、設定 OFF 可) が鳴る
- **Given** オーバーレイ表示
  **When** ESC キー or 閉じるボタン
  **Then** **閉じられない** (本気で終業を迫るため)、ただしダミー勤怠画面の退勤ボタンを押すと閉じる
- **Given** アプリ非アクティブ状態
  **When** 定時が来る
  **Then** Tauri はウィンドウを強制的にフォアグラウンドに戻し、Web はタブのタイトル点滅 + ネイティブ通知で呼び込む
- **Given** 複数端末 (Tauri + Web) で同時ログイン
  **When** 定時
  **Then** 両方で同時にオーバーレイが発火する
- **Given** 効果音は CDN 配信
  **When** 読み込まれる
  **Then** SRI ハッシュ付きで整合性検証される `[SEC-13]`

**INVEST Notes**: Small=△ (2 days、アニメーション含む)

---

## US-03-02: ダミー勤怠画面と退勤ボタン [MVP]

**Story**
> **When** 定時オーバーレイが出ているとき、
> **I want to** 目の前に「退勤」ボタン 1 個のダミー勤怠画面が出てほしい、
> **so I can** 自分の手でクリックして退勤した達成感を得られる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-04-01

**AC**
- **Given** 定時オーバーレイ起動中
  **When** ダミー勤怠画面が中央に出る
  **Then** 「退勤打刻」ボタンが 1 個、大きく表示される
- **Given** 退勤ボタン
  **When** クリックされる
  **Then** 退勤完了画面 (「今日もおつかれ。明日の自分に任せよう」等 AI 生成の一言) が表示される
- **Given** 退勤ボタンが**自動クリックされる** (悪意ある自動化)
  **When** サーバー側で検知
  **Then** 最低限のハートビート/フォーカス確認で抑制を試みる (完璧でなくて OK、デモ要件)
- **Given** 退勤後の状態
  **When** 再度オーバーレイが出ようとする
  **Then** その日 (日付変わるまで) は再発火しない

**INVEST Notes**: Small=○ (1 day)

---

## US-03-03: 早期退勤 (リアクション "もう帰る" 起点) [MVP]

**Story**
> **When** コンテンツを見て「もう帰るぞ」と決意したとき、
> **I want to** 定時前でも退勤動線を自分から起動したい、
> **so I can** 今の勢いのまま退勤ボタンまで行ける。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-04-01, FR-03-02

**AC**
- **Given** コンテンツ提示中
  **When** 「もう帰る」ボタンを押す
  **Then** US-03-01 と同じ全画面オーバーレイが即座に発火
- **Given** 早期退勤オーバーレイ
  **When** ダミー勤怠で打刻
  **Then** リアクション "LEAVE_NOW" が履歴として永続化 `[SEC-03]` (監査ログ、改ざん不可ストレージ) `[SEC-14]`

**INVEST Notes**: Small=○ (0.5 day)

---

## US-03-04: 退勤完了メッセージ (AI 生成) [Full]

**Story**
> **When** 退勤ボタンを押した直後、
> **I want to** その日の自分に合ったねぎらいの一言がほしい、
> **so I can** 気分良く終業できる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-02-01, FR-04-01

**AC**
- **Given** 退勤ボタンクリック
  **When** Bedrock に「今日のプロファイル + 天気 + 見たコンテンツ」を渡す
  **Then** 20-50 文字のねぎらい文が生成・表示される
- **Given** Bedrock がタイムアウト
  **When** 表示寸前
  **Then** テンプレート文言 ("お疲れさま！明日も頑張ろう") にフォールバック `[SEC-15]`

**INVEST Notes**: Small=○ (1 day)

---

# E4. 家族・ペット写真管理

> P3 営業マン + P2 事務の心に刺さる、本プロダクトの感情フック担当。

---

## US-04-01: 写真アップロード (Pre-signed URL) [MVP]

**Story**
> **When** 家族やペットの写真を登録したいとき、
> **I want to** シンプルにドラッグ&ドロップでアップロードしたい、
> **so I can** 「別の未来」に家族写真が混ざるようにできる。

**Persona**: P2 (猫ソラ), P3 (ミケ + 家族)

**Traceability**: FR-05-01, NFR-03 (SECURITY-01, SECURITY-05, SECURITY-08)

**AC**
- **Given** ログイン済みユーザー
  **When** 写真管理画面で画像ファイルをドロップ
  **Then** API Gateway から Pre-signed URL が発行され、クライアントは直接 S3 に PUT する
- **Given** 発行された Pre-signed URL
  **When** 検査する
  **Then** `s3://<bucket>/user/<userId>/photos/<uuid>.ext` にのみ書き込める (他ユーザープレフィックス不可) `[SEC-08]`
- **Given** 許可外拡張子 (例 `.exe`) のアップロード試行
  **When** API が Pre-signed URL 発行要求を受ける
  **Then** 400 で拒否 `[SEC-05]`
- **Given** 10MB 超のファイル
  **When** アップロード試行
  **Then** S3 のバケットポリシー / Pre-signed URL 条件で拒否 `[SEC-05]`
- **Given** 画像メタ情報
  **When** DynamoDB 保存 → 読み出し
  **Then** 完全一致 `[PBT-02]`
- **Given** S3 バケット
  **When** 公開アクセス試行
  **Then** Public Access Block で拒否される `[SEC-09]`

**INVEST Notes**: Small=△ (2 days)

---

## US-04-02: AI キャプション生成 [MVP]

**Story**
> **When** 写真が「別の未来」として選ばれたとき、
> **I want to** ペット/家族の呼び名や現在時刻を踏まえた一言がキャプションで付いてほしい、
> **so I can** 「今日もミケが待ってるよ」と心を動かされる。

**Persona**: P2, P3

**Traceability**: FR-02-02 (家族写真), NFR-05 (PBT)

**AC**
- **Given** 写真が提示コンテンツとして選ばれる
  **When** Bedrock にプロファイル (呼び名) + 時刻 + 画像 ID を渡す
  **Then** 20-60 文字のキャプションが生成される (例「ミケが玄関で待ってる。18:30 までに帰れば夕食も一緒」)
- **Given** キャプション生成の文字数制約
  **When** 100 回繰り返す
  **Then** すべての出力が 20〜60 文字以内 (invariant) `[PBT-03]`
- **Given** 画像 URL
  **When** キャプションに含められる
  **Then** ユーザー自身の画像のみ (IDOR 防止) `[SEC-08]`

**INVEST Notes**: Small=○ (1 day)

---

## US-04-03: 写真一覧・削除 [Full]

**Story**
> **When** 古い写真や見せたくない写真があるとき、
> **I want to** 管理画面から削除したい、
> **so I can** プライバシーを保てる。

**Persona**: P2, P3

**Traceability**: FR-05-01, NFR-08

**AC**
- **Given** 管理画面
  **When** 写真一覧が表示される
  **Then** 自分がアップロードした写真のみ、サムネイルで一覧される
- **Given** 削除ボタン
  **When** クリック
  **Then** S3 オブジェクト + DynamoDB メタが両方削除され、履歴の参照は "削除済" 表示に変わる
- **Given** 他ユーザーの photoId
  **When** 削除 API を叩く
  **Then** 403 `[SEC-08]`
- **Given** 削除操作
  **When** 発生
  **Then** 監査ログに (userId, photoId, action=DELETE, timestamp) が改ざん不可ストレージに書かれる `[SEC-13][SEC-14]`

**INVEST Notes**: Small=○ (1 day)

---

## US-04-04: 写真カテゴリ分類 (家族 / ペット) [Full]

**Story**
> **When** 写真アップロード時、
> **I want to** 「家族」「ペット」のタグを付けたい、
> **so I can** コンテンツの文脈に合わせて選別表示できる。

**Persona**: P2, P3

**Traceability**: FR-05-01

**AC**
- **Given** アップロード時
  **When** タグを選ぶ (family / pet / その他)
  **Then** メタ情報として保存される
- **Given** AI 選定時
  **When** タグを考慮
  **Then** 「家族が待っている」系のプロンプトで family 写真を優先選択

**INVEST Notes**: Small=○ (0.5 day)

---

# E5. プライバシー・セキュリティ

> ユーザーが安心して個人情報を預けられる、透明性の Epic。

---

## US-05-01: 位置情報粒度コントロール [MVP]

**Story**
> **When** Geolocation の許可を求められたとき、
> **I want to** どのレベルの粒度で使われるか説明されたい、
> **so I can** 安心して許可できる。

**Persona**: P1, P2, P3, P4

**Traceability**: FR-02-03, NFR-08

**AC**
- **Given** 初回の位置情報要求
  **When** ブラウザ / Tauri 許可ダイアログが出る前
  **Then** アプリが事前に「区レベル粒度までしか使わない」と説明モーダルを出す
- **Given** 許可後
  **When** 送信する座標
  **Then** 小数第 2 位で丸められている (約 1km 粒度) `[PRV]`
- **Given** ログ
  **When** 確認する
  **Then** 元の精密座標は記録されていない `[SEC-03]`

**INVEST Notes**: Small=○ (0.5 day)

---

## US-05-02: アカウント削除 (データ完全消去) [Full]

**Story**
> **When** 退職・退会したいとき、
> **I want to** 自分のデータをすべて消したい、
> **so I can** 個人情報が残らない安心を得られる。

**Persona**: P1, P2, P3, P4

**Traceability**: NFR-08 (Privacy by Design)

**AC**
- **Given** ログイン済みユーザー
  **When** 設定画面で「アカウント削除」を選ぶ
  **Then** 確認画面 (パスワード再入力) を経て削除が受理される
- **Given** 削除受理
  **When** バッチジョブが走る
  **Then** Cognito ユーザー + DynamoDB の全レコード + S3 の該当プレフィックス画像が削除される
- **Given** 削除操作
  **When** 発生
  **Then** 監査ログ (行為者・対象・時刻) が改ざん不可領域に永続化 `[SEC-14]`
- **Given** 削除後のトークン
  **When** API アクセス
  **Then** 401/403 で拒否

**INVEST Notes**: Small=○ (1-2 days)

---

## US-05-03: セキュリティヘッダー (Web 版) [MVP]

**Story**
> **When** Web 版にブラウザでアクセスしたとき、
> **I want to** モダンなセキュリティヘッダーでサイトが保護されていてほしい、
> **so I can** XSS / クリックジャッキング等から守られる。

**Persona**: P1, P4 (Web 版主利用)

**Traceability**: NFR-03 (SECURITY-04)

**AC**
- **Given** CloudFront から配信される HTML
  **When** レスポンスヘッダーを検査
  **Then** `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin` がすべて付与されている `[SEC-04]`
- **Given** `Content-Security-Policy`
  **When** 検査
  **Then** `unsafe-inline` / `unsafe-eval` を含まない `[SEC-04]`

**INVEST Notes**: Small=○ (0.5 day)

---

## US-05-04: レート制限と異常検知 [Full]

**Story**
> **When** 不審な頻度で API が叩かれたとき、
> **I want to** 自動的に制限がかかってほしい、
> **so I can** 自分のアカウントが悪用されても被害が広がらない。

**Persona**: P1, P2, P3, P4 (全員)

**Traceability**: NFR-03 (SECURITY-11, SECURITY-14)

**AC**
- **Given** API Gateway Usage Plan
  **When** ユーザーあたり 100 req/分 を超える
  **Then** 429 が返る `[SEC-11]`
- **Given** ログイン失敗
  **When** 10 分で 5 回以上
  **Then** 一時ロック + CloudWatch Alarm 発火 `[SEC-12][SEC-14]`

**INVEST Notes**: Small=○ (1 day)

---

## US-05-05: 監査ログ & 通知 [Full]

**Story**
> **When** 重大な操作 (プロファイル変更、写真削除、アカウント削除) があったとき、
> **I want to** 操作の履歴がきちんと残ってほしい、
> **so I can** 問題が起きた時に追跡できる。

**Persona**: P1, P2, P3, P4

**Traceability**: NFR-03 (SECURITY-03, SECURITY-13, SECURITY-14)

**AC**
- **Given** プロファイル変更
  **When** 発生
  **Then** before/after + userId + timestamp + source IP が監査ログに残る `[SEC-13]`
- **Given** 監査ログ書き込み用ロール
  **When** 権限を確認
  **Then** 削除権限がない (append only) `[SEC-14]`
- **Given** CloudWatch
  **When** ログ保持期間を確認
  **Then** 最低 90 日 `[SEC-14]`

**INVEST Notes**: Small=○ (1 day)

---

# Traceability Matrix

## Story → Personas → Requirements (FR/NFR)

| Story | Personas | Requirement |
|---|---|---|
| US-01-01 ユーザー登録 | P1,P2,P3,P4 | FR-01-01, NFR-03 (SEC-12) |
| US-01-02 ログイン | P1,P2,P3,P4 | FR-01-01, NFR-03 (SEC-08, SEC-12), NFR-05 (PBT-02) |
| US-01-03 ヒアリング | P1,P2,P3,P4 | FR-01-02, NFR-08, NFR-05 (PBT-02, PBT-04, PBT-07) |
| US-01-04 定時設定 | P1,P2,P3,P4 | FR-01-03, NFR-05 (PBT-02) |
| US-01-05 プロファイル編集 | P1,P2,P3,P4 | FR-01-02, NFR-03 (SEC-08) |
| US-02-01 スケジューラ | P1,P2,P3,P4 | FR-02-01, NFR-05 (PBT-03, PBT-05, PBT-06) |
| US-02-02 コンテキスト収集 | P1,P2,P3,P4 | FR-02-03, NFR-08, NFR-03 (SEC-03, SEC-15) |
| US-02-03 AI コンテンツ選定 | P1,P2,P3,P4 | FR-02-01, NFR-03 (SEC-03, SEC-05, SEC-15), NFR-05 (PBT-03) |
| US-02-04 コンテンツ提示 UI | P1,P2,P3,P4 | FR-02-04, NFR-03 (SEC-05, SEC-15) |
| US-02-05 リアクション | P1,P2,P3,P4 | FR-03-02, NFR-05 (PBT-04), NFR-03 (SEC-08) |
| US-02-06 飲食店 | P1,P2,P4 | FR-02-02, NFR-05 (PBT-03) |
| US-02-07 趣味系 | P1,P2,P4 | FR-02-02, NFR-05 (PBT-03) |
| US-02-08 電車・天気 | P1,P2,P3 | FR-02-02, NFR-05 (PBT-03) |
| US-02-09 履歴閲覧 | P1,P2,P3,P4 | FR-03-01, NFR-03 (SEC-08) |
| US-03-01 定時オーバーレイ | P1,P2,P3,P4 | FR-04-01, NFR-03 (SEC-13) |
| US-03-02 ダミー勤怠 | P1,P2,P3,P4 | FR-04-01 |
| US-03-03 早期退勤 | P1,P2,P3,P4 | FR-04-01, FR-03-02, NFR-03 (SEC-03, SEC-14) |
| US-03-04 退勤メッセージ | P1,P2,P3,P4 | FR-02-01, FR-04-01, NFR-03 (SEC-15) |
| US-04-01 写真アップロード | P2,P3 | FR-05-01, NFR-03 (SEC-01, SEC-05, SEC-08, SEC-09), NFR-05 (PBT-02) |
| US-04-02 AI キャプション | P2,P3 | FR-02-02, NFR-03 (SEC-08), NFR-05 (PBT-03) |
| US-04-03 写真一覧・削除 | P2,P3 | FR-05-01, NFR-08, NFR-03 (SEC-08, SEC-13, SEC-14) |
| US-04-04 写真カテゴリ | P2,P3 | FR-05-01 |
| US-05-01 位置粒度 | P1,P2,P3,P4 | FR-02-03, NFR-08, NFR-03 (SEC-03) |
| US-05-02 アカウント削除 | P1,P2,P3,P4 | NFR-08, NFR-03 (SEC-14) |
| US-05-03 セキュリティヘッダー | P1,P4 | NFR-03 (SEC-04) |
| US-05-04 レート制限 | P1,P2,P3,P4 | NFR-03 (SEC-11, SEC-12, SEC-14) |
| US-05-05 監査ログ | P1,P2,P3,P4 | NFR-03 (SEC-03, SEC-13, SEC-14) |

## Count Summary

| Epic | MVP Stories | Full Stories | Total |
|---|---|---|---|
| E1 オンボーディング | 4 | 1 | 5 |
| E2 日中のリマインド体験 | 8 | 1 | 9 |
| E3 退勤動線 | 3 | 1 | 4 |
| E4 家族・ペット写真管理 | 2 | 2 | 4 |
| E5 プライバシー・セキュリティ | 2 | 3 | 5 |
| **Total** | **19** | **8** | **27** |

(目標 25〜35 stories 達成)

---

# NFR Compliance Summary

## Security Baseline (NFR-03)
| Rule | Coverage Stories | Status |
|---|---|---|
| SEC-01 暗号化 | US-04-01 他 IaC で保証 | Planned |
| SEC-02 アクセスログ | インフラ層 (今後の Infrastructure Design) | Planned |
| SEC-03 アプリログ | US-02-02, US-02-03, US-03-03, US-05-01, US-05-05 | Compliant |
| SEC-04 HTTP Headers | US-05-03 | Compliant |
| SEC-05 入力検証 | US-01-01, US-01-03, US-01-04, US-02-03, US-02-04, US-04-01 | Compliant |
| SEC-06 最小権限 | IaC (今後) | Planned |
| SEC-07 NW 制限 | IaC (今後) | Planned |
| SEC-08 認可 | US-01-02, US-01-05, US-02-05, US-02-09, US-04-01, US-04-02, US-04-03 | Compliant |
| SEC-09 ハードニング | US-01-01, US-04-01 | Compliant |
| SEC-10 サプライチェーン | CI/CD (今後) | Planned |
| SEC-11 安全な設計 | US-05-04 | Compliant |
| SEC-12 認証 | US-01-01, US-01-02, US-05-04 | Compliant |
| SEC-13 完全性 | US-03-01, US-04-03, US-05-05 | Compliant |
| SEC-14 監視 | US-03-03, US-04-03, US-05-02, US-05-04, US-05-05 | Compliant |
| SEC-15 例外処理 | US-02-02, US-02-03, US-02-04, US-03-04 | Compliant |

## PBT (NFR-05)
| Rule | Coverage Stories | Status |
|---|---|---|
| PBT-01 プロパティ特定 | Functional Design ステージで全 Story を検証予定 | Planned for next stage |
| PBT-02 ラウンドトリップ | US-01-02, US-01-03, US-01-04, US-04-01 | Compliant |
| PBT-03 不変条件 | US-02-01, US-02-03, US-02-06, US-02-07, US-02-08, US-04-02 | Compliant |
| PBT-04 冪等性 | US-01-03, US-02-05 | Compliant |
| PBT-05 オラクル比較 | US-02-01 (スケジューラ参照実装) | Compliant |
| PBT-06 状態 PBT | US-02-01 (提示セッション) | Compliant |
| PBT-07 ジェネレータ品質 | US-01-03 | Compliant |
| PBT-08 シュリンク/再現 | Code Generation で保証 | Planned |
| PBT-09 フレームワーク | NFR-05 で選定済 (Hypothesis / fast-check) | Compliant |
| PBT-10 補完戦略 | Code Generation で example-based も作成 | Planned |

---

# Post-Generation Self-Review (INVEST)

| Criteria | 全体評価 | Notes |
|---|---|---|
| **I** Independent | ◎ | Story 間依存は最小化 (US-02-01 ↔ US-02-04 などのハンドオフは明示済) |
| **N** Negotiable | ○ | AC は具体的だが、実装詳細には踏み込んでいない |
| **V** Valuable | ◎ | 全 Story にペルソナとユーザー価値が紐付く |
| **E** Estimable | ○ | 各 Story の目安工数 (0.5〜3 day) を明記 |
| **S** Small | ◎ | 最大 3 day 規模、2 日超のものは AC で分割可能と注記 |
| **T** Testable | ◎ | 全 AC が Given-When-Then で検証可能、PBT/SEC も具体的 |

結論: 27 Story すべて INVEST を概ね満たす。
