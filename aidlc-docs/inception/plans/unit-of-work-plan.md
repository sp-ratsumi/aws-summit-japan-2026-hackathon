# Unit of Work Plan — アフターファイブ

**Stage**: INCEPTION — Units Generation (Part 1: Planning)
**Purpose**: システムを Unit of Work (独立開発・検証・デプロイ可能な単位) に分解する計画書

---

## 1. 前提コンテキスト

- **Project Type**: Greenfield
- **Frontend**: React + TypeScript + Vite (Tauri + Web PWA の 2 ビルド、Clean Architecture)
- **Backend**: Python on AWS Lambda + Single-Table DynamoDB + Cognito + S3 + Bedrock + CloudFront + API Gateway + EventBridge
- **Service Layer**: 単一 Lambda 内の service 関数 (C1=C)
- **Components**: 25 (FE 9 + BE 12 + Cross-cutting 4)
- **参照**: `aidlc-docs/inception/application-design/*.md`

---

## 2. Unit of Work 候補 (Application Design からの予告)

`component-dependency.md` で予告した 8 Unit 構成を出発点とします:

| 候補 Unit | 含む Components | 主な Stories |
|---|---|---|
| **U1 identity** | BE-01 Auth, BE-02 Profile, S-01 Onboarding, S-06 AccountDeletion | US-01-01, 01-02, 01-03, 01-04, 01-05, 05-02 |
| **U2 reminder** | BE-03 SchedulerRef, BE-04 ContentSelection, BE-05 ContentRepo, BE-07 History, S-02 Reminder | US-02-01, 02-02, 02-03, 02-06, 02-07, 02-08, 02-09 |
| **U3 termination** | BE-08 Reaction, BE-09 Termination, S-03 Termination, S-04 Reaction | US-02-05, 03-01, 03-02, 03-03, 03-04 |
| **U4 photo** | BE-06 Photo, S-05 PhotoUpload | US-04-01, 04-02 (with U2), 04-03, 04-04 |
| **U5 audit-observability** | BE-11 Audit, X-01 Observability | US-05-04, 05-05 (partial) |
| **U6 shared-library** | X-02 SharedModels, X-03 RepositoryLayer, X-04 ErrorHandling | 横断 (Python layer) |
| **U7 frontend** | FE-01〜FE-09 (Tauri + Web 両ビルド) | 全フロント Story |
| **U8 infrastructure** | BE-12 InfrastructureComponent (CDK) | インフラ横断 (US-05-03 セキュリティヘッダー等) |

---

## 3. Terminology

- **Unit of Work (UOW)**: 開発 / 設計 / テスト / コードレビュー の粒度。CONSTRUCTION Phase の per-unit ループの単位。
- **Service**: 本プロジェクトでは論理上 Lambda 関数群の集まりに相当。単一デプロイ可能性は Infrastructure Design で確定。
- **Module**: Unit 内の論理サブディビジョン (例: U2 reminder 内の BE-03/04/05/07)

**Note**: 本プロジェクトはすべての Backend を 1 Lambda デプロイにまとめる選択肢も、Unit ごとに Lambda 分割する選択肢もあり得る。**Unit は "開発粒度" としての分解であり、デプロイ粒度は Infrastructure Design で確定する**。

---

## 4. Planning Questions

**回答方法**: 各質問の `[Answer]:` タグの後に A/B/C/D/X を記入してください。

### Question U1: Unit の数
Unit の数は?

A) **予告通り 8 Unit** (U1〜U8、バランス型)
B) **6 Unit に集約**: U3 と U4 を U2 に統合 (reminder 周りを大きく)、U5/U6 を U8 に統合 — シンプル、per-unit ステージが短い
C) **10 Unit に細分**: U7 frontend を "onboarding UI" / "reminder UI" / "termination UI" / "photo UI" に分ける + U8 infra を "api-infra" / "frontend-infra" に分割
D) **5 Unit に大胆集約**: `core-backend` (U1+U2+U3+U5) + `photo` (U4) + `shared-lib` (U6) + `frontend` (U7) + `infra` (U8)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question U2: Unit 境界の基準
Unit 境界を決める主軸は?

A) **機能ドメイン単位** (identity / reminder / termination / photo など、上記候補の切り方)
B) **デプロイアーティファクト単位** (Lambda 関数 / Tauri アプリ / Web SPA / CDK ソースツリー で分ける)
C) **チーム分担単位** (hackathon 個人戦なら 1 人の 1 日分でキリの良い粒度)
D) **トランザクション境界単位** (冪等性や consistency の境界ごと)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question U3: Shared Library Unit (U6) の位置付け
`shared-library` (X-02/X-03/X-04) はどう扱いますか?

A) **独立 Unit** として per-unit ループに入れる (Functional Design から Code Generation まで順次)
B) **U1 (identity) の一部に含める** — identity が最初に開発されるので、その中で shared 基盤を作る
C) **Unit の外の "ゼロ次ステップ"** として最初に作り、per-unit ループには入れない
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question U4: Infrastructure Unit (U8) の位置付け
`infrastructure` (CDK) はどう扱いますか?

A) **独立 Unit U8** として扱い、他 Unit と並行開発可能な形で進める
B) **各 Unit の Infrastructure Design ステージで、その Unit 分のスタックを書く** (U8 を設けず分散)
C) **U1 に bootstrap infra を置き、以降は Unit 毎に追記** (共通 Network / Cognito / DynamoDB Table の初期化は U1、各機能の Lambda / S3 bucket は各 Unit 側)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question U5: Unit 開発順序
Unit 開発順序を何で決めますか?

A) **依存関係順** (U6 shared → U8 infra bootstrap → U1 identity → U2 reminder → U3 termination → U4 photo → U5 audit → U7 frontend)
B) **ユーザー価値順** (U7 frontend のプロトタイプ → U2 reminder コア → U1 identity → 周辺)
C) **ハッカソン MVP / Full 優先度順** (MVP Story を含む Unit を先に、Full のみを含む Unit を後に)
D) **並列実行を最大化** (U6/U8 先行 → 残り同時並行)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question U6: 並列開発の許容度
per-unit ループを並列に進められますか?

A) **Yes (フル並列)** — 依存解消後はすべて並列、AI エージェントが別セッションで分担
B) **No (直列)** — 1 Unit ずつ順番に完成させる、ハッカソンでは現実的
C) **部分並列** — "shared-lib / infra-bootstrap" 完成後、残りを 2〜3 並列
X) Other (please describe after [Answer]: tag below)

[Answer]: C

### Question U7: Frontend Unit の分割
Frontend Unit (U7) を単一にしますか、分割しますか?

A) **単一 U7-frontend** — FE 9 components を 1 Unit で扱う (上記予告通り)
B) **機能別に分割**: U7a-onboarding-ui / U7b-reminder-ui / U7c-termination-ui / U7d-photo-ui / U7e-shell (共通 AppShell + ApiClient + PlatformAdapter)
C) **技術別に分割**: U7-core-ui (React コンポーネント) + U7-tauri-shell (Tauri Rust 側) + U7-web-shell (PWA manifest 等)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question U8: Code Organization (Greenfield, multi-unit)
Greenfield + 複数 Unit の場合のディレクトリ構造は?

A) **Monorepo (単一リポジトリ) with pnpm/turbo workspaces + Python Poetry monorepo-style**:
   ```
   /
   ├── apps/
   │   ├── desktop/       (Tauri shell)
   │   ├── web/           (Web PWA)
   │   └── backend/       (Lambda functions, CDK)
   ├── packages/
   │   ├── shared-ts/     (フロント共通、U6 の一部)
   │   └── shared-py/     (backend 共通、U6)
   ├── infrastructure/    (CDK, U8)
   ├── openapi/           (schema.yaml, API 契約)
   └── aidlc-docs/
   ```
B) **Polyrepo (分割リポジトリ)**: frontend / backend / infrastructure の 3 リポ
C) **Flat monorepo**: apps / packages を使わず、フラット構成
X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Question U9: Deployment Model
バックエンドの Lambda デプロイモデルは?

A) **モノリシック Lambda** — 1 Lambda 関数に全ハンドラ、API Gateway のルーティングで分岐
B) **関数分割 (Route 1:1)** — エンドポイントごとに独立 Lambda (例: `/profile` 用, `/content/next` 用...)
C) **Unit 単位で 1 Lambda** — Unit ごとに 1 Lambda、内部で複数ルートを handle
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## 5. Unit 生成の Mandatory Artifacts (回答後に実行する)

- [x] `aidlc-docs/inception/application-design/unit-of-work.md` — Unit 定義 + 責務 + Components + Stories + コード配置
- [x] `aidlc-docs/inception/application-design/unit-of-work-dependency.md` — Unit 間依存マトリクス + 開発順序 + 並列化計画
- [x] `aidlc-docs/inception/application-design/unit-of-work-story-map.md` — Story ↔ Unit ↔ Priority (MVP/Full) マッピング
- [x] Unit 境界検証 (すべての Story が 1 以上の Unit に、循環依存なし)

---

## 6. Generation Sub-Plan (実行済み)

- [x] Step A: 回答をもとに Unit 数と境界を確定 (8 Unit / 機能ドメイン)
- [x] Step B: `unit-of-work.md` 生成 (各 Unit の definition, responsibility, components, stories, directory, tech)
- [x] Step C: `unit-of-work-dependency.md` 生成 (マトリクス, 開発順, 並列化)
- [x] Step D: `unit-of-work-story-map.md` 生成 (MVP 19 / Full 8)
- [x] Step E: Self-review (全 25 components が 1 Unit に所属 / 全 27 stories が 1 以上の Unit に / 循環依存なし)
- [x] Step F: aidlc-state.md 更新 + audit.md 記録
