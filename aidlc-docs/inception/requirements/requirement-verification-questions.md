# Requirements Verification Questions — アフターファイブ

このプロジェクト「アフターファイブ」の要件を明確化するため、以下の質問にお答えください。

**回答方法**: 各質問の `[Answer]:` タグの後に、選択肢の文字 (A, B, C, D, E, X) を記入してください。「Other」を選んだ場合は、説明を追記してください。

---

## コンテキスト / ハッカソン

### Question 1
このプロジェクトの主目的は何ですか?

A) AWS Summit Japan 2026 ハッカソン用のデモ/プロトタイプ (動くものをデモできれば良い)
B) ハッカソンで評価される本格的な MVP (AWS サービスをしっかり活用する)
C) ハッカソン後も継続して開発・運用する本番サービスの出発点
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question 2
開発期間および完成させたいスコープは?

A) 数日以内、主要機能1〜2個に絞った「刺さるデモ」
B) 1〜2週間、コア機能すべてを浅く広く実装
C) 2週間以上、すべての機能を丁寧に実装
X) Other (please describe after [Answer]: tag below)

[Answer]: X MVPは2週間、ちゃんとしたデモは1ヶ月、すべての機能を丁寧に実装

### Question 3
想定する審査観点 / 売り (いちばん推したい点) は?

A) アイデアの面白さ・ユーモア (「人をダメにする」コンセプト性)
B) AWS AI サービス (Bedrock など) を活用したパーソナライズ
C) 位置情報/外部 API 連携などの技術的面白さ
D) 実用性・ユーザーの課題解決力
X) Other (please describe after [Answer]: tag below)

[Answer]: X ビジネス意図（Intent）の明確さ、創造性とテーマ適合性、Unit 分解の適切さ、ドキュメントの品質。ポイントはアイデアと技術のバランスが重要。アイデアがよくても設計やドキュメントが悪いと高得点は難しい

---

## プラットフォーム / 形態

### Question 4
アプリケーションの形態は?

A) Web アプリ (PC ブラウザ中心、デモしやすい)
B) モバイルアプリ (iOS/Android ネイティブ) - 位置情報や通知と相性が良い
C) モバイル対応 Web アプリ (PWA、レスポンシブ)
D) ブラウザ拡張 (仕事画面に「ポップアップ」で侵食してくるイメージに合う)
X) Other (please describe after [Answer]: tag below)

[Answer]: X デスクトップアプリ　Tauri

### Question 5
ユーザー認証・登録はどうしますか?

A) 認証不要 (デモ用途、セッション内だけの状態管理)
B) シンプルなメール/パスワード認証 (Amazon Cognito)
C) SNS ログイン (Google など) を Cognito 経由で
D) ハッカソンではダミーユーザー、将来的に Cognito で本実装
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## コア機能: 定時1時間前の「別の未来」提示

### Question 6
「定時」の判定方法は?

A) ユーザーが自分の定時 (例: 18:00) を設定する (シンプル)
B) 曜日別に定時を設定できる (柔軟)
C) 勤怠システムから自動取得 (今回は範囲外)
D) デモ用に「ボタン1つで定時1時間前を疑似発火」できる仕組みも併せて用意
X) Other (please describe after [Answer]: tag below)

[Answer]: A 設定で変更可能（デフォルト18:00）

### Question 7
「別の未来」コンテンツは 1 回にいくつ提示しますか? また、どう選びますか?

A) 毎回 1 個だけ、ユーザーの登録情報と現在のコンテキスト (天気・時間等) からAI が選ぶ
B) 3〜5 個をカード形式で並べて表示
C) フィード形式で次々スクロールして見られる
X) Other (please describe after [Answer]: tag below)

[Answer]: X 17時とかは1回で1コンテンツ出したりするが、18時になるにつれて高頻度でコンテンツが流れたりするようにする

### Question 8
AI によるパーソナライズ (コンテンツ選定・文言生成) に使う技術は?

A) Amazon Bedrock (Claude 等) でコンテンツ選定 + 煽り文言を動的生成
B) シンプルなルールベース (条件分岐) のみ、AI は使わない
C) Bedrock + Amazon Personalize の組み合わせ
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 個別コンテンツの機能スコープ

### Question 9
「家族・ペット写真」機能の実装範囲は?

A) 画像アップロード + 定期的にランダム表示 (S3 保存)
B) アップロード + AI でキャプション/煽り文言生成 ("今日も〇〇が待ってるよ")
C) ハッカソンではスコープ外 (実装しない)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question 10
「近隣の飲食店 (居酒屋・ラーメン・カフェ)」機能の実装範囲は?

A) 位置情報 + Google Places API (または Hot Pepper API) で近隣取得、画像付きで表示
B) 位置情報 + ダミーデータで表示 (外部 API 未利用、ハッカソン簡易)
C) スコープ外
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question 11
「サウナ・ジム・映画・アニメ・推し配信・ゲーム情報」の実装範囲は?

A) すべてのカテゴリを実装 (各々外部 API 連携)
B) 1〜2 カテゴリに絞る (デモ映えするもの、例: 映画上映情報 + ゲームセール)
C) 事前に仕込んだダミーデータで雰囲気だけ作る
D) スコープ外
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question 12
「電車の混雑・遅延」情報の実装範囲は?

A) 鉄道会社の公開 API / 遅延情報フィードを組み込み
B) ダミーデータで演出
C) スコープ外
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question 13
「天気」情報 (これから雨なので帰れ、等) の実装範囲は?

A) OpenWeatherMap 等の API を利用、位置情報から自動取得
B) 固定/ダミーデータ
C) スコープ外
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question 14
位置情報はどのように取得しますか?

A) ブラウザ Geolocation API (ユーザー許可で取得)
B) ユーザーが最寄り駅/地域を登録 (許可不要、プライバシー配慮)
C) 両方選べる (デフォルトは B、許可があれば A)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## ユーザーヒアリング (初回のみ)

### Question 15
初回のヒアリングで収集すべき項目は? (複数選択 OK、最も近いものを)

A) 最小構成: 定時・最寄り地域・趣味カテゴリ (2〜3 問で完結)
B) 中構成: 上記 + 家族/ペットの有無、使用路線、好きなゲームジャンル (5〜7 問)
C) 大構成: 上記 + 詳細な趣味・推し・食の好みまで (10 問以上)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question 16
ヒアリング結果の活用方法は?

A) AI (Bedrock) へのプロンプトに含め、コンテンツ選定に使う
B) DB (DynamoDB) に保存し、ルールベースでコンテンツ絞り込みに使う
C) 両方: DB に保存 + AI プロンプトにも利用
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 終了動線 (勤怠入力画面の提示)

### Question 17
「勤怠入力システム画面」はどう実装しますか?

A) 実在する勤怠システム (例: ジョブカン、KING OF TIME) の API 連携
B) それっぽいダミー勤怠画面をアプリ内に実装し、「打刻しました!」と表示
C) 指定した URL (ユーザー設定の勤怠 URL) を全画面で開く
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question 18
「勤怠画面が目の前に現れる」演出の強度は?

A) 控えめ: 画面上部に通知バナー
B) 標準: モーダルで中央に表示
C) 派手: 全画面オーバーレイ + SE/アニメーションで「帰るぞ!」
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## 非機能・インフラ (AWS)

### Question 19
AWS のどのサービス構成を想定しますか? (ハッカソン向けに最適なもの)

A) サーバーレス中心: CloudFront + S3 (フロント) + API Gateway + Lambda + DynamoDB + Bedrock
B) コンテナ: ECS Fargate + ALB + RDS + Bedrock
C) Amplify (フル活用で高速構築)
D) おまかせ (AI 側で最適構成を提案してほしい)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question 20
フロントエンドの技術スタックは?

A) React + TypeScript + Vite
B) Next.js (React + SSR)
C) Vue.js / Nuxt
D) おまかせ
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question 21
バックエンドの言語・フレームワークは?

A) Node.js + TypeScript (Lambda)
B) Python (Lambda) - Bedrock SDK との親和性
C) Go (Lambda)
D) おまかせ
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question 22
通知機能 (推しの配信開始通知など) の実装範囲は?

A) Web Push 通知 (ブラウザ)
B) アプリ内通知のみ (起動中のみ表示)
C) スコープ外 (時限表示だけにする)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## データ / 画像アップロード

### Question 23
家族・ペット写真のアップロード / 保存要件は?

A) S3 直接アップロード (Pre-signed URL)、メタ情報は DynamoDB
B) API Gateway 経由でアップロード
C) スコープ外
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question 24
データの永続化が必要な項目は? (いちばん近いもの)

A) 最小: ユーザープロファイル (ヒアリング結果) + アップロード画像のみ
B) 上記 + コンテンツ提示履歴 (いつ何を見せたか)
C) 上記 + ユーザーのリアクション (見た/無視した/帰った) の学習データ
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## ブランディング / UI

### Question 25
UI のトーンは?

A) ユーモラス・煽り系 (「今帰らないと雨に濡れるぞ」「推しが配信始まったぞ」)
B) 落ち着いた癒やし系 (そっと日常を思い出させる)
C) 両方 (コンテンツごとに使い分け、AI が文言生成)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question 26
言語対応は?

A) 日本語のみ
B) 日本語 + 英語 (ハッカソン審査向け)
C) 多言語対応
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 拡張機能 (Opt-In)

### Question: Security Extensions
Should security extension rules be enforced for this project?

A) Yes — enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)
B) No — skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question: Property-Based Testing Extension
Should property-based testing (PBT) rules be enforced for this project?

A) Yes — enforce all PBT rules as blocking constraints (recommended for projects with business logic, data transformations, serialization, or stateful components)
B) Partial — enforce PBT rules only for pure functions and serialization round-trips (suitable for projects with limited algorithmic complexity)
C) No — skip all PBT rules (suitable for simple CRUD applications, UI-only projects, or thin integration layers with no significant business logic)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## その他

### Question 27
その他、絶対に入れたい機能・逆に絶対に外したい機能、もしくは譲れないこだわりがあれば教えてください (自由記述)。

[Answer]: 特になし
