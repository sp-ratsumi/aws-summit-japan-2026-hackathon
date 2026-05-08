# アフターファイブ

> **明日の自分に任せて今日は精一杯ダメになる**

「人をダメにするサービス」をテーマに、**仕事人間を "ダメモード" へ堕とす** ことで翌日の創造性を引き出すプロダクト。
AWS Summit Japan 2026 Hackathon 向けに、[AI-DLC (AI-Driven Development LifeCycle)](https://github.com/aws-samples/sample-aws-ai-dlc) を使って構築している。

---

## コンセプト

<img src="aidlc-docs/inception/assets/concept.png" alt="アフターファイブのコンセプト図" />

### "ダメにする" の意味

世間的には「生産性を下げる」と呼ばれがちな **ダメな欲望** — 家族・推し・食・酒・趣味・ペット — こそが、明日の創造性の源であるという仮説に全賭けしている。
仕事モードに居残る働き手に、定時 1 時間前から **ダメな未来** を視界に差し込み、定時 = **堕落ゲート** で退勤動線に叩き込む。

### 3 つの体験装置

| # | 装置 | 役割 |
|---|---|---|
| 🪜 **堕落ランプ** | 17:00 → 18:00 にかけてコンテンツ提示頻度が逓増する時間窓 | 仕事モードを段階的に解体 |
| 🪞 **ダメな未来ジェネレータ** | Amazon Bedrock (Claude) がユーザーの "ダメな欲望プロファイル" を踏まえて日本語の煽り文言を生成 | 堕落を後押し |
| 🚪 **堕落ゲート** | 定時に発動する全画面オーバーレイ + ダミー勤怠画面 | 「今日の仕事は終わりだ、明日の自分に任せてダメになれ」と本人に宣言させる |

### ダメな欲望カテゴリ (D1〜D7)

| # | カテゴリ | MVP |
|---|---|:---:|
| D1 | 家族・ペット | ✅ |
| D2 | 推し (配信・新曲) | ✅ |
| D3 | 食事 (飯テロ) | ✅ |
| D4 | 酒 (ビール/サワー/ハイボール/日本酒/ワイン) | ✅ |
| D5 | 趣味 (サウナ/ジム/映画/アニメ/ゲーム) | ✅ |
| D6 | 通勤/天気 | ✅ |
| D7 | 睡眠・休息 | Full |

詳細は [requirements.md](aidlc-docs/inception/requirements/requirements.md) を参照。

---

## ターゲットペルソナ

| ペルソナ | 帰れない理由 | 刺さるダメな欲望 |
|---|---|---|
| P1 田中 健太 (32) 残業 SE | タスク管理の責任感 | D5 サウナ / D4 ビール / D6 天気 |
| P2 佐藤 美咲 (27) 趣味事務職 | 「キリの良いところまで」病 | D2 推し / D5 アニメ / D3 カフェ / D4 ワイン |
| P3 鈴木 翔 (38) ペット飼い営業 | 顧客対応の切れ目がない | D1 猫 + 家族 / D3 夕食 / D4 ハイボール |
| P4 山田 リョウ (25) ゲーマー推し活エンジニア | スプリント末の駆け込み | D5 ゲーム / D2 推し新曲 / D4 サワー |

詳細は [personas.md](aidlc-docs/inception/user-stories/personas.md) を参照。

---

## アーキテクチャ

```
┌──────────────────────────────────┐
│  Client Tier (同一 React + TS)    │
│  ├─ Tauri Desktop (macOS/Win)    │   堕落ランプ / 堕落ゲート描画
│  └─ Web PWA (CloudFront + S3)    │
└──────────────┬───────────────────┘
               │ HTTPS + Cognito JWT
┌──────────────▼───────────────────┐
│  API & Compute Tier              │
│  API Gateway (REST)              │
│  └─ AWS Lambda (Python 3.12)     │   ダメな未来ジェネレータ
│     + Lambda Authorizer          │
└──────────────┬───────────────────┘
               │
┌──────────────▼───────────────────┐
│  Data & AI Tier                  │
│  ├─ DynamoDB (Single-Table)      │   ダメな欲望プロファイル / 履歴
│  ├─ S3 (Photos, SSE-KMS, OAC)    │   D1 家族・ペット素材
│  ├─ Amazon Cognito User Pool     │
│  ├─ Amazon Bedrock (Claude)      │   堕落煽りコピー生成
│  ├─ EventBridge                  │   監査 / 通知イベント
│  └─ CloudWatch                   │
└──────────────────────────────────┘
```

詳細は [application-design.md](aidlc-docs/inception/application-design/application-design.md) を参照。

---

## 技術スタック

| 領域 | 技術 |
|---|---|
| Frontend | React 18 + TypeScript + Vite |
| Desktop Shell | Tauri v2 (Rust) |
| Web Distribution | Amazon CloudFront + S3 (OAC) |
| Backend | Python 3.12 on AWS Lambda + Lambda Powertools |
| API | Amazon API Gateway (REST) |
| Auth | Amazon Cognito User Pool |
| Database | Amazon DynamoDB (Single-Table) |
| Object Storage | Amazon S3 (SSE-KMS + Public Access Block) |
| AI | Amazon Bedrock (Claude) |
| IaC | AWS CDK (Python) |
| Test (Py) | pytest + Hypothesis (PBT) |
| Test (TS) | Vitest + fast-check (PBT) |

---

## リポジトリ構成 (予定)

```
aws-summit-japan-2026-hackathon/
├── apps/
│   ├── desktop/                 # Tauri shell (Rust + React)
│   ├── web/                     # Web PWA (Vite)
│   └── backend/                 # Lambda 関数群 (U1〜U5)
│       └── functions/
│           ├── identity/        # U1: /profile*, /auth/*, /account/*
│           ├── reminder/        # U2: /content/*, /sessions/*, /history
│           ├── termination/     # U3: /reactions, /terminations/*
│           ├── photo/           # U4: /photos/*
│           ├── audit/           # U5: EventBridge consumer
│           └── authorizer/      # Lambda Authorizer
├── packages/
│   ├── frontend-core/           # 共通 React (feature-sliced)
│   ├── shared-ts/               # OpenAPI 由来の型 + util
│   └── shared-py/               # Lambda Layer (Pydantic/Repo/Auth/...)
├── infrastructure/              # AWS CDK (Python)
├── openapi/
│   └── schema.yaml              # API 契約 (Single source of truth)
├── aidlc-docs/                  # AI-DLC 生成ドキュメント
└── README.md
```

---

## 開発プロセス (AI-DLC)

本プロジェクトは AI-DLC ワークフローに沿って段階的に成果物を積み上げている。現在の進捗:

### 🔵 INCEPTION フェーズ — 完了済み

- [x] Workspace Detection
- [x] Requirements Analysis — [requirements.md](aidlc-docs/inception/requirements/requirements.md)
- [x] User Stories — [stories.md](aidlc-docs/inception/user-stories/stories.md) / [personas.md](aidlc-docs/inception/user-stories/personas.md)
- [x] Workflow Planning
- [x] Application Design — [application-design.md](aidlc-docs/inception/application-design/application-design.md)
- [x] Units Generation — [unit-of-work.md](aidlc-docs/inception/application-design/unit-of-work.md)

### 🟢 CONSTRUCTION フェーズ — 進行中

8 Unit を以下の順序で Per-Unit Loop (Functional Design → NFR Requirements → NFR Design → Infrastructure Design → Code Generation) に投入する:

| Phase | Units |
|---|---|
| Phase 0 (基盤 / 直列) | U6 shared-library → U8 infrastructure (bootstrap) |
| Phase 1 (並列可) | U1 identity / U2 reminder / U4 photo |
| Phase 2 | U3 termination (U2 依存) / U5 audit-observability |
| Phase 3 | U7 frontend (Phase 1 の API 契約 lock 後) |

### 🟡 OPERATIONS フェーズ — 未着手 (プレースホルダ)

---

## スコープ

| 指標 | 数 |
|---|---:|
| User Story | 28 (MVP 20 / Full 8) |
| Component | 25 (Frontend 9 / Backend 12 / Cross-cutting 4) |
| Service | 6 |
| Unit of Work | 8 |

### MVP スコープ外 (ダミー or Full 送り)

- 外部 API 連携 (Google Places / Hot Pepper / OpenWeatherMap / 鉄道遅延 / 推し配信 / Steam 等) はすべてダミーデータ
- 実在の勤怠システム連携 — **アプリ内ダミー勤怠画面 (堕落ゲート) のみ**
- SNS ログイン (メール/パスワードのみ)
- 多言語対応 (日本語のみ)
- モバイルネイティブアプリ (Web PWA で代替)
- D7 睡眠・休息カテゴリ (Full)
- 「翌日の創造性」の定量化 (コンセプト仮説として据え置き)

---

## セキュリティ / 品質

AI-DLC の 2 つのエクステンションを **Full mode (全ルール blocking)** で有効化している:

- **Security Baseline** (15 ルール): 暗号化 / 最小権限 IAM / CSP・HSTS / IDOR 多層防御 / Cognito JWT 検証 / 監査ログ / fail-closed 例外 など
- **Property-Based Testing** (10 ルール): Hypothesis (Python) + fast-check (TS)、堕落ランプの単調非増加不変条件 / 冪等性 / round-trip / stateful PBT など

詳細は [requirements.md §3 NFR-03/NFR-05](aidlc-docs/inception/requirements/requirements.md) を参照。

---

## ドキュメント

- [要件定義](aidlc-docs/inception/requirements/requirements.md)
- [ペルソナ](aidlc-docs/inception/user-stories/personas.md) / [ユーザーストーリー](aidlc-docs/inception/user-stories/stories.md)
- [アプリケーション設計](aidlc-docs/inception/application-design/application-design.md)
  - [コンポーネント](aidlc-docs/inception/application-design/components.md)
  - [コンポーネントメソッド](aidlc-docs/inception/application-design/component-methods.md)
  - [サービス](aidlc-docs/inception/application-design/services.md)
  - [依存関係](aidlc-docs/inception/application-design/component-dependency.md)
  - [Unit of Work](aidlc-docs/inception/application-design/unit-of-work.md)
- [AI-DLC state tracking](aidlc-docs/aidlc-state.md)
- [audit log](aidlc-docs/audit.md)

---

## ライセンス

未定 (ハッカソン作品)。

---

> _今日ダメになる自由 → 明日の創造性_
