# 要件定義書

## 概要（Introduction）

アフターファイブ（定時誘惑サービス）は、企業で働く従業員が「定時までのあと1時間をどう過ごすか」という一日の最後の意思決定ポイントを消し去り、定時での退勤を自然な流れとして発生させることを目的とした B2B SaaS である。

本サービスは、定時の一定時間前に仕事と無関係な魅力的なコンテンツ（近隣店舗、ゲーム、配信、映像、フードデリバリー、天気、交通など）を従業員のディスプレイ上に提示し、定時到来時には勤怠管理システム（jinjer）の退勤画面 URL への画面遷移を行う。これにより「帰るかどうか」「どう過ごすか」という個人の判断を事前に削ぎ落とし、「帰る」という実行のみを残す。

本バージョン（v2）では、スコープを次のとおり明確化している。

- **jinjer 連携は画面遷移のみ**（API 連携や打刻完了検知は行わない）
- **Teams 等の外部チャットサービスとの連携は行わない**（「周囲を巻き込む誘い文」機能はスコープ外）
- **誘引コンテンツは単一の「オファー情報」ではなく、17 カテゴリの Nudge_Content として扱う**

ビジネスモデルとしては、労働時間を厳守させたい企業が従業員向けに導入する形態を想定する。AWS Summit Japan 2026 ハッカソンの提出作品として開発される。

## 用語集（Glossary）

### システムコンポーネント

- **After_Five_System**: 本要件定義全体を指すシステム。以降の各サブシステムを包含する。
- **Nudge_Orchestrator**: 時刻・スケジュール状態に応じて各ディスプレイ・通知を統合的に制御するコンポーネント。
- **Nudge_Content_Display**: 従業員のデスクトップ／ブラウザ上に Nudge_Content を提示する UI コンポーネント。
- **Nudge_Content_Catalog**: 全カテゴリの Nudge_Content を格納・管理するデータストアおよび取得インターフェース。
- **Exit_Screen_Navigator**: 定時到来時に勤怠管理画面（jinjer）の退勤画面 URL への画面遷移を実行するコンポーネント。API 連携や打刻完了検知は行わない。
- **Admin_Console**: 導入企業の管理者が対象従業員・定時・コンテンツ配信条件などを設定する管理画面。
- **Content_Provider**: Nudge_Content を Nudge_Content_Catalog に提供する外部データ供給元（飲食店舗、クーポンサービス、ゲームプラットフォーム、配信サービス、天気情報 API、交通情報 API 等）。

### データ・属性

- **Employee_Profile**: 従業員ごとの定時、勤務地、勤務区分、オプトイン状態などを保持する情報。
- **Tenant**: 本サービスを導入する企業単位の論理境界。
- **Scheduled_End_Time**: Employee_Profile に登録された従業員個別の定時（例: 18:00）。
- **Pre_End_Window**: Scheduled_End_Time から一定時間前の区間（定時の 60 分前から定時到来まで）。
- **Opt_Out_State**: 従業員が自らの意思で誘引コンテンツの表示を停止している状態。
- **Analytics_Report**: 導入効果（定時退勤率、コンテンツ閲覧率等）を Admin 向けに提示するレポート。

### ユーザー

- **Employee**: 導入企業に所属し、本サービスの対象となるエンドユーザ。
- **Admin**: 導入企業側で本サービスの設定・運用を行う管理者ユーザ。

### Nudge_Content カテゴリ

**Nudge_Content_Category**: 誘引コンテンツの種別。以下の列挙値を持つ。

| 列挙値 | カテゴリ名 | 内容例 |
|---|---|---|
| LOCAL_IZAKAYA | 近隣の居酒屋 | 徒歩圏の居酒屋の割引・限定情報 |
| LOCAL_RAMEN | 近隣のラーメン店 | 徒歩圏のラーメン店情報 |
| LOCAL_CAFE | 近隣のカフェ | 徒歩圏のカフェ情報 |
| LOCAL_SAUNA | 近隣のサウナ | 徒歩圏のサウナ・スパ情報 |
| LOCAL_GYM | 近隣のジム | 徒歩圏の当日ドロップイン可能ジム情報 |
| LOCAL_CINEMA | 近隣の映画館 | 徒歩圏の映画館の当日上映スケジュール |
| GAMING_DAILY_QUEST | ゲームのデイリークエスト | プレイ中のゲームの未消化デイリー情報 |
| GAMING_FRIEND_ONLINE | フレンドのログイン状況 | ゲームプラットフォーム上のフレンドのオンライン状態 |
| GAMING_STREAMER_LIVE | 推しの配信開始通知 | 登録済みストリーマー／VTuber の配信開始予定・開始通知 |
| GAMING_SALE | セール中のゲーム | ゲームストアの期間限定セール情報 |
| GAMING_LIMITED_EVENT | 今日だけのイベント | 当日限定のゲーム内イベント情報 |
| GAMING_BACKLOG_RESUME | 積みゲーのおすすめ再開ポイント | 進行中断中のゲームタイトルと次に遊ぶべき推奨ポイント |
| HOME_MEDIA | 家で観られる映画・アニメ | ストリーミングサービスで視聴可能な作品 |
| FOOD_DELIVERY | フードデリバリーのクーポン | フードデリバリーサービスの当日有効クーポン |
| FITNESS_SCHEDULE | ジムのスケジュール | 所属ジムのレッスン・予約枠情報 |
| COMMUTE_TRAIN | 電車の込み具合 | 最寄り路線の混雑度・遅延情報 |
| WEATHER_ALERT | 天気（帰宅促進） | 降雨・荒天など「今のうちに帰る」理由となる気象情報 |

### カテゴリグループ（同一の振る舞いを持つカテゴリの分類）

- **LOCAL_VENUE_GROUP**: LOCAL_IZAKAYA、LOCAL_RAMEN、LOCAL_CAFE、LOCAL_SAUNA、LOCAL_GYM、LOCAL_CINEMA。実店舗を伴うカテゴリであり、勤務地からの徒歩時間を評価対象とする。
- **DIGITAL_CONTENT_GROUP**: GAMING_DAILY_QUEST、GAMING_FRIEND_ONLINE、GAMING_STREAMER_LIVE、GAMING_SALE、GAMING_LIMITED_EVENT、GAMING_BACKLOG_RESUME、HOME_MEDIA。従業員個人のアカウント状態やコンテンツプロバイダのリアルタイム状態に依存するカテゴリ。
- **CONVENIENCE_GROUP**: FOOD_DELIVERY、FITNESS_SCHEDULE。当日時点で有効なクーポン／スケジュール情報を持つカテゴリ。
- **CONTEXT_SIGNAL_GROUP**: COMMUTE_TRAIN、WEATHER_ALERT。従業員の判断材料となる外部コンテキスト（混雑・天気）を提示するカテゴリ。

## 要件（Requirements）

### Requirement 1: 定時前の別の未来の提示

**User Story:** 導入企業の Employee として、定時前の最後の1時間に自動的に仕事と無関係な魅力的なコンテンツが視界に入ることで、「残り時間をどう埋めるか」を自分で判断せずに済む状態になりたい。

#### Acceptance Criteria

1. WHEN 現在時刻が Employee の Scheduled_End_Time の60分前に到達したとき、THE Nudge_Orchestrator SHALL 当該 Employee に対して Nudge_Content_Display の表示処理を起動する。
2. WHEN Nudge_Orchestrator から Nudge_Content_Display の表示処理が起動されたとき、THE Nudge_Content_Display SHALL Nudge_Content_Catalog から取得した Nudge_Content を当該 Employee のディスプレイ上に表示する。
3. THE Nudge_Content_Display SHALL 1 度の表示において Nudge_Content_Category のうち少なくとも 3 以上の異なるカテゴリのコンテンツを混在させて表示する。
4. THE Nudge_Content_Display SHALL 各 Nudge_Content について、カテゴリ名、タイトル（店舗名・ゲーム名・作品名・路線名等の主題）、カテゴリ固有の要約情報（徒歩時間／割引率／配信者名／遅延分数等）、有効期限または鮮度タイムスタンプを表示する。
5. WHERE Nudge_Content が LOCAL_VENUE_GROUP に属する場合、THE Nudge_Content_Display SHALL 勤務地からの所要徒歩時間を併記する。
6. WHERE Nudge_Content が WEATHER_ALERT カテゴリに属する場合、THE Nudge_Content_Display SHALL 現在地周辺の降雨予報時刻と降水強度の目安を含めた帰宅促進メッセージを表示する。
7. WHERE Nudge_Content が COMMUTE_TRAIN カテゴリに属する場合、THE Nudge_Content_Display SHALL 最寄り路線の遅延有無および混雑度の目安を表示する。
8. WHILE Nudge_Content_Display がコンテンツを表示している間、THE Nudge_Content_Display SHALL 「今すぐ予約」「今すぐ行く」「今すぐ購入」「今すぐログイン」等、業務の即時中断を要求するコールトゥアクション要素を含まない表示のみを行う。
9. IF Nudge_Content_Catalog から当該 Employee に提示可能な Nudge_Content を 1 件も取得できないとき、THEN THE Nudge_Content_Display SHALL 表示処理をスキップし、その事実を Nudge_Orchestrator に通知する。
10. WHERE Employee_Profile の Opt_Out_State が有効に設定されている場合、THE Nudge_Orchestrator SHALL 当該 Employee に対する Nudge_Content_Display の表示処理を起動しない。

### Requirement 2: Nudge_Content_Catalog による多カテゴリコンテンツの管理と取得

**User Story:** 本サービスの運用者として、実店舗情報・ゲーム関連情報・映像情報・デリバリークーポン・気象／交通情報などの多様な誘引コンテンツを一元管理し、Employee の勤務地・嗜好・当日状況に応じて適切なコンテンツを提示できるようにしたい。

#### Acceptance Criteria

1. THE Nudge_Content_Catalog SHALL 各 Nudge_Content について、コンテンツ識別子、Nudge_Content_Category、タイトル、要約本文、提供元 Content_Provider 識別子、取得日時、鮮度有効期限を保持する。
2. WHERE Nudge_Content が LOCAL_VENUE_GROUP に属する場合、THE Nudge_Content_Catalog SHALL 当該 Nudge_Content について店舗の位置情報（緯度・経度）と営業時間を追加で保持する。
3. WHERE Nudge_Content が DIGITAL_CONTENT_GROUP に属する場合、THE Nudge_Content_Catalog SHALL 当該 Nudge_Content について対象プラットフォーム識別子（例: Steam、PlayStation、Twitch、YouTube、Netflix 等のコード値）と、紐づく外部コンテンツ識別子を追加で保持する。
4. WHERE Nudge_Content が CONVENIENCE_GROUP に属する場合、THE Nudge_Content_Catalog SHALL 当該 Nudge_Content についてクーポンコードまたは予約枠識別子と適用条件（金額下限、時間帯等）を追加で保持する。
5. WHERE Nudge_Content が CONTEXT_SIGNAL_GROUP に属する場合、THE Nudge_Content_Catalog SHALL 当該 Nudge_Content について対象地点（路線／エリア）と有効時間窓（何時から何時までの予報・混雑情報か）を追加で保持する。
6. WHEN Nudge_Content_Display が Employee の勤務地・プラットフォーム連携状態を伴って Nudge_Content を要求したとき、THE Nudge_Content_Catalog SHALL 当該 Employee に適合する Nudge_Content（LOCAL_VENUE_GROUP は徒歩 15 分以内、DIGITAL_CONTENT_GROUP は Employee が連携済みのプラットフォームに限る、CONTEXT_SIGNAL_GROUP は当該時刻の有効範囲内）のみを返却する。
7. WHEN Nudge_Content_Catalog が Content_Provider から Nudge_Content を受領したとき、THE Nudge_Content_Catalog SHALL カテゴリごとの必須項目（Criterion 1〜5 参照）の存在を検証する。
8. IF Content_Provider から受領した Nudge_Content の必須項目が欠落しているとき、THEN THE Nudge_Content_Catalog SHALL 当該 Nudge_Content を登録せず、検証エラーを監査ログに記録する。
9. WHEN Nudge_Content の鮮度有効期限が現在時刻を経過したとき、THE Nudge_Content_Catalog SHALL 当該 Nudge_Content を検索結果から除外する。

### Requirement 3: 定時到来時の退勤動線の可視化（jinjer 画面遷移）

**User Story:** 導入企業の Employee として、定時になった瞬間に退勤操作の画面が目の前に現れることで、「帰るかどうか」の判断を行うことなく退勤打刻という実行だけを行いたい。

#### Acceptance Criteria

1. WHEN 現在時刻が Employee の Scheduled_End_Time に到達したとき、THE Exit_Screen_Navigator SHALL 当該 Employee のデフォルトブラウザまたは指定クライアントアプリにおいて jinjer の退勤操作画面 URL を最前面タブまたはウィンドウとして開く。
2. THE Exit_Screen_Navigator SHALL jinjer の退勤画面 URL を開く動作のみを行い、退勤打刻操作の代行および jinjer の API 連携による打刻完了検知は行わない。
3. IF Exit_Screen_Navigator が jinjer の退勤画面 URL の起動に失敗したとき、THEN THE Exit_Screen_Navigator SHALL 代替案内として当該 URL をクリップボードに複製した上で通知領域にリンクを提示し、失敗事象を監査ログに記録する。
4. WHERE Employee_Profile の当日の勤務区分が在宅勤務に設定されている場合、THE Exit_Screen_Navigator SHALL 退勤画面 URL の自動前面表示を行わず、通知領域への URL 提示のみを行う。
5. WHEN Exit_Screen_Navigator が退勤画面 URL の表示処理を一度実行したとき、THE Nudge_Orchestrator SHALL 当該 Employee に対する当日の Nudge_Content_Display の新規起動を停止する。

### Requirement 4: 従業員プロファイルと定時設定

**User Story:** 導入企業の Admin として、従業員ごとに定時や勤務区分を登録し、個別の勤務実態に即した誘引処理が行われるようにしたい。

#### Acceptance Criteria

1. THE Admin_Console SHALL Admin に対して Employee_Profile の新規作成、更新、削除の操作を提供する。
2. THE Employee_Profile SHALL 従業員識別子、所属 Tenant、Scheduled_End_Time、勤務地（緯度経度またはオフィス識別子）、当日の勤務区分（出社／在宅／休暇のいずれか）、Opt_Out_State、連携済み外部プラットフォーム識別子の一覧を保持する。
3. WHEN Admin が Employee_Profile に Scheduled_End_Time を登録したとき、THE Admin_Console SHALL 入力値が 00:00 から 23:59 の範囲内かつ分単位で指定されていることを検証する。
4. IF Admin が登録した Scheduled_End_Time が上記検証条件に合致しないとき、THEN THE Admin_Console SHALL 登録処理を中止し、検証エラーメッセージを Admin に表示する。
5. WHERE Employee_Profile の当日の勤務区分が休暇に設定されている場合、THE Nudge_Orchestrator SHALL 当該 Employee に対する当日の全ての誘引処理（Nudge_Content_Display、Exit_Screen_Navigator）を起動しない。
6. THE Admin_Console SHALL Employee 本人に対して、連携済み外部プラットフォーム識別子の追加・削除の自主操作を提供する。

### Requirement 5: 従業員のオプトアウト

**User Story:** 導入企業の Employee として、本サービスによる誘引処理を自らの判断で停止できるようにし、意図しない情報提示による業務妨害を回避したい。

#### Acceptance Criteria

1. THE After_Five_System SHALL Employee に対して Opt_Out_State の有効化および無効化の切替操作を提供する。
2. WHEN Employee が Opt_Out_State を有効化したとき、THE Nudge_Orchestrator SHALL 当該 Employee に対する以降の全ての誘引処理（Nudge_Content_Display、Exit_Screen_Navigator の自動前面表示）を停止する。
3. WHEN Employee が Opt_Out_State を無効化したとき、THE Nudge_Orchestrator SHALL 次回の Pre_End_Window 到来以降、当該 Employee に対する誘引処理を再開する。
4. THE After_Five_System SHALL Employee の Opt_Out_State の変更履歴を、変更日時と変更主体識別子とともに記録する。

### Requirement 6: 導入企業単位のマルチテナント分離

**User Story:** 導入企業の Admin として、自社の従業員データおよび設定が他社から参照・改変されないように、企業ごとに論理的に分離された環境で本サービスを利用したい。

#### Acceptance Criteria

1. THE After_Five_System SHALL 全てのデータ（Employee_Profile、Nudge_Content_Catalog の企業別カスタマイズ、Analytics_Report）を Tenant 識別子に紐付けて保持する。
2. WHEN Admin または Employee がデータを参照する要求を行ったとき、THE After_Five_System SHALL 要求者の所属 Tenant 識別子と一致するデータのみを返却する。
3. IF Admin が自身の所属 Tenant と異なる Tenant のデータに対する参照または更新を要求したとき、THEN THE After_Five_System SHALL 当該要求を拒否し、アクセス拒否事象を監査ログに記録する。

### Requirement 7: 導入効果レポート

**User Story:** 導入企業の Admin として、本サービスの導入効果を定量的に確認し、労働時間遵守の改善状況を経営層に報告できるようにしたい。

#### Acceptance Criteria

1. THE Analytics_Report SHALL Tenant 単位で、集計期間内の対象 Employee 数、Nudge_Content_Display の提示回数、Nudge_Content_Category 別の提示比率、Exit_Screen_Navigator の起動回数、Opt_Out_State の有効化率を集計する。
2. WHEN Admin が集計期間を指定して Analytics_Report の生成を要求したとき、THE Admin_Console SHALL 指定期間に対応する Analytics_Report を生成して表示する。
3. THE Analytics_Report SHALL 個々の Employee を特定可能な形式ではなく、集計値または匿名化済みの値として表示する。
4. IF Admin が集計期間として未来日付または Tenant 契約開始日以前の日付を指定したとき、THEN THE Admin_Console SHALL 当該期間の指定を拒否し、検証エラーメッセージを Admin に表示する。

### Requirement 8: スケジュールベースの発火制御

**User Story:** 導入企業の Employee として、休暇日や勤務時間外には誘引処理が発生しないようにし、業務外時間における意図しない情報提示を避けたい。

#### Acceptance Criteria

1. WHILE 現在時刻が Employee の Scheduled_End_Time の 60 分前よりも前である間、THE Nudge_Orchestrator SHALL 当該 Employee に対する Nudge_Content_Display を起動しない。
2. WHILE 現在時刻が Employee の Scheduled_End_Time を過ぎて 1 時間を経過した以降である間、THE Nudge_Orchestrator SHALL 当該 Employee に対する新規の誘引処理（Nudge_Content_Display、Exit_Screen_Navigator）を起動しない。
3. WHEN 現在時刻が Employee の Scheduled_End_Time を過ぎたとき、AND Exit_Screen_Navigator の起動が当日まだ行われていないとき、THE Nudge_Orchestrator SHALL Scheduled_End_Time 経過後 15 分間隔で Exit_Screen_Navigator を起動し、最大 4 回（すなわち Scheduled_End_Time から 60 分以内）まで再試行する。
4. WHERE 当日が Tenant カレンダー上の非営業日（休日、企業指定休業日）に該当する場合、THE Nudge_Orchestrator SHALL 当該日の全ての誘引処理を起動しない。
