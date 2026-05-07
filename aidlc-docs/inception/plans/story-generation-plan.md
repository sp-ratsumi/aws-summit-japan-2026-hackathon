# Story Generation Plan — アフターファイブ

**Stage**: INCEPTION — User Stories (Part 1: Planning)
**Purpose**: ユーザーストーリー・ペルソナ生成の方針と具体的な手順を決める計画書

---

## 1. Story Breakdown Approach (提案)

候補アプローチと、本プロジェクトでの推奨:

| Approach | 内容 | 本プロジェクトへの適合度 |
|---|---|---|
| User Journey-Based | ユーザーの 1 日の流れ (出社→17時→18時→退勤) に沿って並べる | **◎** — コアの "1 日ストーリー" が鮮明になる、ハッカソン審査の説明と相性良し |
| Feature-Based | 機能 (Auth / Profile / Scheduler / ...) 単位で並べる | ○ — FR との対応が取りやすい |
| Persona-Based | ペルソナごとにストーリーを分ける | △ — 本プロジェクトは個別ペルソナで行動が分岐しないため重複が出やすい |
| Domain-Based | ビジネスドメインで分ける | △ — 単一ドメイン (働き手の退勤支援) |
| Epic-Based | Epic > Story の階層 | ○ — 大きめの単位 (Epic) を軸にした方が管理しやすい |

**推奨**: **Epic-Based + User Journey-Based のハイブリッド**
- 上位: 5〜7 個の Epic (オンボーディング / 日中のリマインド体験 / 退勤動線 / 画像管理 / マルチチャネル / 認証・プライバシー)
- 下位: Epic ごとに 3〜6 個の User Story を INVEST 基準で記述
- Story は可能な限り User Journey の流れで整列

---

## 2. Planning Checklist (実行前に決める事項)

以下の質問を解決して、ストーリー生成に取りかかります。

- [ ] Q1. ストーリー記述フォーマット (Connextra vs Job Story vs ...)
- [ ] Q2. ペルソナの体数と切り口
- [ ] Q3. Epic の分割粒度 (5〜7 Epic か、もっと細かく / 粗く)
- [ ] Q4. Acceptance Criteria のフォーマット (Given-When-Then / Checklist)
- [ ] Q5. ストーリーの優先度ラベル付けの有無 (MoSCoW / Must-Have-Only)
- [ ] Q6. Non-Functional Requirements (セキュリティ、PBT 等) をストーリー化するか、Requirements に留めるか
- [ ] Q7. 追加ペルソナ特性 (職業・生活パターン・推しの有無等)

---

## 3. Mandatory Artifacts (生成するファイル)

- [ ] `aidlc-docs/inception/user-stories/personas.md` — ペルソナ定義
- [ ] `aidlc-docs/inception/user-stories/stories.md` — ユーザーストーリー (Epic + Story + AC)
- [ ] 各ストーリーが INVEST (Independent / Negotiable / Valuable / Estimable / Small / Testable) を満たす
- [ ] 各ストーリーに Acceptance Criteria を付与
- [ ] ペルソナ → ストーリーのマッピング
- [ ] Requirements (FR-XX) → Story (US-XX) のトレーサビリティ

---

## 4. Questions for Planning (please answer)

**回答方法**: `[Answer]:` タグの後に A / B / C / D / X と記入してください。

### Question 1: ストーリー記述フォーマット
採用するストーリー記述フォーマットは?

A) **Connextra 形式** ("As a <user>, I want <goal>, so that <benefit>")
B) **Job Story 形式** ("When <situation>, I want to <motivation>, so I can <outcome>")
C) **ハイブリッド** — 基本は Connextra、UX ストーリーのみ Job Story (トリガーの文脈が重要な場合)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question 2: ペルソナの構成
ペルソナは何体、どのような切り口で作成しますか?

A) **2 体**: 残業しがちな SE / 趣味で帰りたい事務職 の 2 軸
B) **3 体**: A の 2 体 + ペット飼い (家族写真機能の主使用者) — 家族系ストーリーを厚く描きたい場合
C) **4 体**: A + ペット飼い + ゲーマー推し活勢 (趣味系コンテンツのメイン受益者)
D) **1 体のみ**: 代表ペルソナ 1 人で十分 (シンプル)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question 3: Epic 分割粒度
Epic の粒度は?

A) **推奨: 6 Epic**
    1. オンボーディング (認証 + 初回ヒアリング + 定時設定)
    2. 日中のリマインド体験 (スケジューラ + コンテンツ提示 + AI 生成)
    3. 退勤動線 (定時オーバーレイ + ダミー勤怠画面)
    4. 家族・ペット写真管理 (アップロード + キャプション生成)
    5. マルチチャネル配信 (Tauri + Web)
    6. プライバシー・セキュリティ (認証 / 位置情報粒度 / 監査)
B) **4 Epic** (認証 / 体験 / 退勤 / 管理) 、マルチチャネル・セキュリティは横断要素としてストーリー化せず Requirements 参照
C) **8 Epic 以上** (通知・履歴・リアクション等も独立 Epic に)
X) Other (please describe after [Answer]: tag below)

[Answer]: X マルチチャネル配信以外

### Question 4: Acceptance Criteria のフォーマット
各ストーリーの AC (受け入れ基準) の書き方は?

A) **Given-When-Then (Gherkin)** — テスト (E2E / PBT) とトレーサビリティが取りやすい
B) **Bullet Checklist** — シンプル、軽量
C) **併用**: Given-When-Then をメインにし、補足として Bullet Checklist
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question 5: 優先度ラベル
ストーリーに優先度ラベルを付けますか?

A) **MoSCoW** (Must / Should / Could / Won't) を付与 — MVP 絞り込みに有効
B) **MVP 2 週 / フル 1 ヶ月** の 2 段階ラベル — ユーザーの回答 (Q2 "MVP 2週、デモ 1ヶ月") に合わせる
C) **優先度ラベルなし** — Workflow Planning で決める
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Question 6: NFR のストーリー化
NFR (Security / PBT / パフォーマンス) はストーリー化しますか?

A) **Requirements に留め、Story には入れない** — NFR は全 Story 横断の制約
B) **"Security / Privacy" Epic を 1 個用意し、主要 NFR のユーザー価値を Story 化** (例: 「他人の写真が見えない」「サービス停止時の代替文言」)
C) **各機能 Story の AC に NFR を織り込む**
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question 7: ペルソナ属性の詳細
ペルソナに含めるべき属性は? 次の中で必須にするものは?

A) **基本のみ**: 名前・年齢・職業・主要ペイン・主要ゲイン
B) **中程度**: A + 1 日のタイムライン (出社〜帰宅) + IT リテラシー + 使用デバイス
C) **詳細**: B + 家族構成 + 趣味・推し・ゲームジャンル + 典型的な残業パターン (物語調)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question 8: ストーリー総数の目安
ストーリー総数の目安は?

A) **軽量 (15〜20 stories)** — MVP にフォーカス、Workflow Planning で取捨選択
B) **標準 (25〜35 stories)** — MVP + デモ 1ヶ月分までカバー
C) **網羅 (40+ stories)** — 将来拡張も含めて網羅的に
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## 5. Generation Sub-Plan (回答後に実行する手順)

以下は回答承認後、Part 2 (Generation) で順に実行する checklist です。現時点ではすべて未着手 [ ]。

- [x] Step A: 上記回答をもとにペルソナを作成し `personas.md` に書き出す
- [x] Step B: Epic 一覧 (Q3 の回答に基づく) を `stories.md` の冒頭で定義
- [x] Step C: Epic ごとに User Story (Q1 フォーマット) を生成、AC (Q4 フォーマット) を付与
- [x] Step D: 各 Story に 優先度 (Q5) を付与 (指定時のみ)
- [x] Step E: Story → Persona → Requirement (FR-XX) のトレーサビリティ表を末尾に添付
- [x] Step F: INVEST チェックリストで self-review、未達 Story を修正
- [x] Step G: NFR (Security / PBT) の反映方針 (Q6) を `stories.md` 内に明示
