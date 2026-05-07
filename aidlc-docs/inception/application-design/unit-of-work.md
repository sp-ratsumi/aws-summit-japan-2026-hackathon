# Unit of Work — アフターファイブ

**Version**: 1.0
**Last Updated**: 2026-05-07
**Decomposition Strategy**: 機能ドメイン単位 (U2=A)、8 Unit (U1=A)

---

## Summary

本プロジェクトは **8 Unit** に分解されます。各 Unit は CONSTRUCTION Phase の per-unit ループ (Functional Design → NFR Requirements → NFR Design → Infrastructure Design → Code Generation) の単位です。

| Unit | Name | 主責務 | MVP Story 数 | Full Story 数 |
|---|---|---|---:|---:|
| **U1** | identity | 認証・プロファイル・アカウント削除 | 4 | 2 |
| **U2** | reminder | スケジューラ参照実装・コンテンツ選定・履歴 | 7 | 1 |
| **U3** | termination | リアクション・終了動線 | 4 | 1 |
| **U4** | photo | 画像アップロード・配信・管理 | 2 | 2 |
| **U5** | audit-observability | 監査ログ・可観測性 | 2 | 2 |
| **U6** | shared-library | 共通 Python Lambda Layer + TS 共有型 | 0 (横断) | 0 (横断) |
| **U7** | frontend | Tauri + Web PWA (React/TS) | 全 MVP 対応 | 全 Full 対応 |
| **U8** | infrastructure | CDK (Python) 全スタック | (横断) | (横断) |

---

## Repository Structure (Monorepo, U8=A)

```
aws-summit-japan-2026-hackathon/
├── apps/
│   ├── desktop/                   # U7 - Tauri shell (Rust + React)
│   │   ├── src-tauri/             # Tauri Rust 側
│   │   ├── src/                   # 共通 React (packages/frontend-core を参照)
│   │   ├── public/
│   │   └── tauri.conf.json
│   ├── web/                       # U7 - Web PWA (Vite)
│   │   ├── src/                   # 共通 React (packages/frontend-core を参照)
│   │   ├── public/
│   │   ├── manifest.webmanifest
│   │   └── vite.config.ts
│   └── backend/                   # U1〜U5 + BE-10 Notification 用 Lambda 関数群
│       ├── functions/
│       │   ├── identity/          # U1: /profile*, /auth/*, /account/*
│       │   │   ├── post_profile.py
│       │   │   ├── get_profile.py
│       │   │   ├── patch_profile.py
│       │   │   ├── delete_account.py
│       │   │   └── pre_signup_trigger.py
│       │   ├── reminder/          # U2: /content/*, /sessions/*, /history
│       │   │   ├── post_content_next.py
│       │   │   ├── get_history.py
│       │   │   ├── post_session_start.py
│       │   │   └── post_session_stop.py
│       │   ├── termination/       # U3: /reactions, /terminations/*
│       │   │   ├── post_reaction.py
│       │   │   ├── post_clock_out.py
│       │   │   └── event_leave_now_consumer.py
│       │   ├── photo/             # U4: /photos/*
│       │   │   ├── post_upload_url.py
│       │   │   ├── post_confirm_upload.py
│       │   │   ├── get_photos.py
│       │   │   └── delete_photo.py
│       │   ├── audit/             # U5: EventBridge consumer
│       │   │   └── audit_event_consumer.py
│       │   └── authorizer/        # Lambda Authorizer (共通)
│       │       └── jwt_authorizer.py
│       └── pyproject.toml
├── packages/
│   ├── frontend-core/             # U7 - 共通 React コンポーネント/hooks/adapters
│   │   ├── src/
│   │   │   ├── features/          # Feature (onboarding, reminder, termination, photo)
│   │   │   ├── shared/            # AppShell, ApiClient, PlatformAdapter
│   │   │   ├── adapters/          # Tauri/Web 実装切替
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── shared-ts/                 # U6 - フロント共通 (OpenAPI 由来の型生成 + 共通 util)
│   │   ├── src/
│   │   │   ├── generated/         # openapi-typescript 生成 (commit しない案もあり)
│   │   │   └── utils/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── shared-py/                 # U6 - バックエンド共通 (Lambda Layer で配布)
│       ├── afterfive_shared/
│       │   ├── models/            # Pydantic モデル (X-02 SharedModels)
│       │   ├── repositories/      # X-03 RepositoryLayer (Profile, Photo, ...)
│       │   ├── errors/            # X-04 ErrorHandling (例外階層)
│       │   ├── observability/     # X-01 Lambda Powertools wrapper
│       │   ├── auth/              # BE-01 AuthComponent (verify_jwt, @requires_ownership)
│       │   ├── clients/           # Bedrock / EventBridge 共通クライアント
│       │   └── __init__.py
│       └── pyproject.toml
├── infrastructure/                # U8 - AWS CDK (Python)
│   ├── stacks/
│   │   ├── auth_stack.py          # Cognito
│   │   ├── data_stack.py          # DynamoDB + S3 + KMS
│   │   ├── api_stack.py           # API Gateway + Lambda + Authorizer
│   │   ├── frontend_stack.py      # CloudFront + S3 for web bundle
│   │   ├── observability_stack.py # CloudWatch + EventBridge + Audit Lambda
│   │   └── event_bus_stack.py     # EventBridge bus + rules
│   ├── app.py                     # CDK entry
│   ├── cdk.json
│   └── pyproject.toml
├── openapi/
│   └── schema.yaml                # API 契約 (Single source of truth)
├── aidlc-docs/                    # ドキュメント
├── pnpm-workspace.yaml
├── turbo.json
├── poetry.lock                    # monorepo Python root (必要であれば)
├── pyproject.toml                 # ルート (tooling、uv/poetry)
├── package.json
└── README.md
```

---

## Unit Definitions

---

### U1: identity

- **Purpose**: ユーザー認証・プロファイル・アカウント削除
- **Components**: BE-01 Auth, BE-02 Profile, S-01 Onboarding, S-06 AccountDeletion
- **Directory**:
  - Backend: `apps/backend/functions/identity/`
  - OpenAPI: `openapi/schema.yaml` の `/auth`, `/profile`, `/account` パス
- **Lambda Functions** (U9=B Route 1:1):
  - `POST /profile` → `post_profile.py`
  - `GET /profile` → `get_profile.py`
  - `PATCH /profile` → `patch_profile.py`
  - `DELETE /account` → `delete_account.py`
  - Cognito Pre-Signup trigger → `pre_signup_trigger.py`
  - `jwt_authorizer.py` (Lambda Authorizer, 全 API 共通)
- **External AWS**: Cognito User Pool, DynamoDB (AfterFiveTable), EventBridge
- **Stories Covered**: US-01-01, US-01-02, US-01-03, US-01-04, US-01-05, US-05-02
- **MVP / Full**: 4 MVP + 2 Full (Full=US-01-05 プロファイル編集, US-05-02 アカウント削除)
- **Key Tech**:
  - Python 3.12
  - Cognito via `boto3 cognito-idp`
  - Pydantic v2 for validation
- **Depends On**: U6 (shared-py), U8 (infra bootstrap)

---

### U2: reminder

- **Purpose**: コア体験 — 「別の未来」コンテンツ選定とプッシュ、履歴管理
- **Components**: BE-03 SchedulerRef, BE-04 ContentSelection, BE-05 ContentRepo, BE-07 History, S-02 Reminder
- **Directory**:
  - Backend: `apps/backend/functions/reminder/`
  - OpenAPI: `/content/next`, `/sessions/*`, `/history` パス
- **Lambda Functions**:
  - `POST /content/next` → `post_content_next.py` (コア)
  - `GET /history?days=N` → `get_history.py`
  - `POST /sessions/start` → `post_session_start.py`
  - `POST /sessions/stop` → `post_session_stop.py`
- **External AWS**: Amazon Bedrock (Claude), DynamoDB, EventBridge
- **Stories Covered**: US-02-01, 02-02, 02-03, 02-06, 02-07, 02-08, 02-09, (US-04-02 AI キャプションは U2 にも依存)
- **MVP / Full**: 7 MVP + 1 Full (Full=US-02-09 履歴閲覧)
- **Key Tech**:
  - Bedrock client (`boto3 bedrock-runtime`)
  - Hypothesis PBT for SchedulerRef / invariants
  - Dummy data seeding for ContentRepo (`content-seed.json`)
- **Depends On**: U6 (shared-py), U8 (infra bootstrap), U4 (PhotoRepository for family/pet images - read-only access)

---

### U3: termination

- **Purpose**: リアクション記録 + 退勤動線 (定時オーバーレイ後のサーバー側処理)
- **Components**: BE-08 Reaction, BE-09 Termination, S-03 Termination, S-04 Reaction
- **Directory**:
  - Backend: `apps/backend/functions/termination/`
  - OpenAPI: `/reactions`, `/terminations/*` パス
- **Lambda Functions**:
  - `POST /reactions` → `post_reaction.py`
  - `POST /terminations/clock-out` → `post_clock_out.py`
  - EventBridge consumer → `event_leave_now_consumer.py` (reaction.leave_now)
- **External AWS**: DynamoDB, EventBridge, Bedrock (ねぎらい文生成は U2 の BE-04 経由で呼び出し)
- **Stories Covered**: US-02-05 (リアクション), US-03-01 (オーバーレイ起動は FE, バックエンドは受け側), US-03-02, US-03-03, US-03-04
- **MVP / Full**: 4 MVP + 1 Full (Full=US-03-04 退勤メッセージ AI 生成)
- **Depends On**: U6, U8, **U2** (ContentSelection の closing message 生成を再利用)

---

### U4: photo

- **Purpose**: 家族・ペット写真のアップロード・配信・管理
- **Components**: BE-06 Photo, S-05 PhotoUpload
- **Directory**:
  - Backend: `apps/backend/functions/photo/`
  - OpenAPI: `/photos/*` パス
- **Lambda Functions**:
  - `POST /photos/upload-url` → `post_upload_url.py`
  - `POST /photos/{photoId}/confirm` → `post_confirm_upload.py`
  - `GET /photos` → `get_photos.py`
  - `DELETE /photos/{photoId}` → `delete_photo.py`
- **External AWS**: S3 (PhotoBucket, SSE-KMS, Public Access Block), DynamoDB, EventBridge, KMS
- **Stories Covered**: US-04-01, US-04-02 (BE-04 とジョイント), US-04-03, US-04-04
- **MVP / Full**: 2 MVP (US-04-01, 02) + 2 Full (US-04-03 削除, US-04-04 タグ分類)
- **Depends On**: U6, U8

---

### U5: audit-observability

- **Purpose**: 監査ログと可観測性 (EventBridge consumer + 共通 Logger middleware)
- **Components**: BE-11 Audit, X-01 Observability
- **Directory**:
  - Backend: `apps/backend/functions/audit/`
  - 共通部分は `packages/shared-py/afterfive_shared/observability/`
- **Lambda Functions**:
  - EventBridge consumer → `audit_event_consumer.py` (全 audit event を受けて AuditTable へ append)
- **External AWS**: DynamoDB AuditTable, CloudWatch Logs, EventBridge
- **Stories Covered**: US-05-04 (レート制限 alarming), US-05-05 (監査ログ)
- **MVP / Full**: 0 MVP / 2 Full (ただし alarm と log retention 設定は U8 インフラで MVP から実装)
- **Depends On**: U6, U8

---

### U6: shared-library

- **Purpose**: 横断的な Python Lambda Layer と TypeScript 共通型
- **Components**: X-01 Observability, X-02 SharedModels, X-03 RepositoryLayer, X-04 ErrorHandling, BE-01 AuthComponent のコード本体
- **Directory**:
  - Python Lambda Layer: `packages/shared-py/afterfive_shared/`
  - TypeScript 共通: `packages/shared-ts/`
  - OpenAPI: `openapi/schema.yaml` (source of truth)
- **Deliverables**:
  - Lambda Layer `afterfive-shared-layer-v1` を CDK で作成
  - npm package `@afterfive/shared-ts` (workspace リンク)
  - `afterfive_shared` パッケージ (内部 Python パッケージ、PyPI には公開しない)
  - OpenAPI → Pydantic 自動生成スクリプト
  - OpenAPI → TypeScript 自動生成スクリプト (`openapi-typescript`)
- **Stories Covered**: 横断 (全 Unit から利用)
- **MVP / Full**: 全 MVP で利用
- **Key Tech**:
  - Pydantic v2 (Python)
  - AWS Lambda Powertools for Python
  - `openapi-typescript` (frontend 型生成)
- **Depends On**: なし (**Unit 依存グラフの起点**)

---

### U7: frontend

- **Purpose**: Tauri デスクトップ + Web PWA のフロントエンド実装全体
- **Components**: FE-01〜FE-09 (AppShell, Onboarding, ReminderFeed, NotificationAdapter, TerminationOverlay, PhotoManager, SchedulerClient, ApiClient, PlatformAdapter)
- **Directory**:
  - 共通 React: `packages/frontend-core/` (feature-sliced: onboarding, reminder, termination, photo)
  - Tauri shell: `apps/desktop/` (Rust + React entry)
  - Web PWA shell: `apps/web/` (Vite + React entry)
- **Stories Covered**: 全 27 Story のクライアントサイド実装 (US-05-03 セキュリティヘッダーは U8 で提供、FE 側は CSP 遵守のコード記述のみ)
- **MVP / Full**:
  - MVP: 全フロント Story のクライアント実装
  - Full: 追加フィーチャ (履歴閲覧 UI, 写真削除 UI, アカウント削除 UI 等)
- **Key Tech**:
  - React 18 + TypeScript
  - Vite (Web + Tauri 共通ビルダー)
  - Tauri v2 (Rust side)
  - `@tauri-apps/api`, `@tauri-apps/plugin-notification`
  - State: Zustand or TanStack Query (Functional Design で決定)
  - Testing: Vitest + fast-check (PBT) + Playwright (E2E)
  - Adapter 実装: `TauriNotificationAdapter` / `WebNotificationAdapter` / `TauriPlatformAdapter` / `WebPlatformAdapter`
- **Depends On**: U6 (shared-ts の OpenAPI 型), U8 (API エンドポイント URL を環境変数から取得)

---

### U8: infrastructure

- **Purpose**: AWS CDK (Python) による全スタックの定義
- **Components**: BE-12 InfrastructureComponent
- **Directory**: `infrastructure/`
- **Stacks**:
  - **AuthStack**: Cognito User Pool + App Client + Pre-Signup Lambda
  - **DataStack**: DynamoDB `AfterFiveTable` + `AuditTable` + S3 `PhotoBucket` + KMS CMK + S3 `WebBundleBucket`
  - **ApiStack**: API Gateway REST + Lambda Authorizer + Usage Plan + Response integration
  - **FrontendStack**: CloudFront Distribution + OAC + Response Headers Policy (CSP/HSTS 他)
  - **EventBusStack**: EventBridge Bus + Rules (user.onboarded, profile.updated, reaction.leave_now_requested, user.terminated, photo.uploaded, photo.deleted, account.deletion.*)
  - **ObservabilityStack**: CloudWatch Dashboards + Alarms + Log Groups + Audit Lambda
- **Stories Covered**: US-05-03 (セキュリティヘッダー), US-05-04 (レート制限), US-05-05 (監査基盤)
- **MVP / Full**:
  - MVP: AuthStack, DataStack, ApiStack (最小), FrontendStack, EventBusStack
  - Full: ObservabilityStack の高度なダッシュボード
- **Key Tech**:
  - AWS CDK v2 (Python)
  - AWS Lambda Python Runtime 3.12
  - AWS WAF (Full 向け、MVP は未使用 or 最小)
- **Depends On**: **なし直接** (他 Unit のコード成果物 (zip) を参照するが、設計時点の依存は U6 のみ)
- **Bootstrap Requirement**: 他 Unit の Lambda 配布前に、AuthStack + DataStack + ApiStack スケルトン (空 Lambda でも) が必要

---

## Component Allocation Summary

| Component | Unit |
|---|---|
| FE-01 AppShell | U7 |
| FE-02 OnboardingModule | U7 |
| FE-03 ReminderFeedModule | U7 |
| FE-04 NotificationAdapter | U7 |
| FE-05 TerminationOverlayModule | U7 |
| FE-06 PhotoManagerModule | U7 |
| FE-07 SchedulerClient | U7 |
| FE-08 ApiClient | U7 |
| FE-09 PlatformAdapter | U7 |
| BE-01 AuthComponent | U1 (コード) + U6 (Layer 化) |
| BE-02 ProfileComponent | U1 |
| BE-03 SchedulerComponent (Ref) | U2 |
| BE-04 ContentSelectionComponent | U2 |
| BE-05 ContentRepositoryComponent | U2 |
| BE-06 PhotoComponent | U4 |
| BE-07 HistoryComponent | U2 |
| BE-08 ReactionComponent | U3 |
| BE-09 TerminationComponent | U3 |
| BE-10 NotificationComponent | U3 (実装は最小、Full で拡張) |
| BE-11 AuditComponent | U5 |
| BE-12 InfrastructureComponent | U8 |
| X-01 Observability | U6 |
| X-02 SharedModels | U6 |
| X-03 RepositoryLayer | U6 |
| X-04 ErrorHandling | U6 |
| S-01 OnboardingService | U1 |
| S-02 ReminderService | U2 |
| S-03 TerminationService | U3 |
| S-04 ReactionService | U3 |
| S-05 PhotoUploadService | U4 |
| S-06 AccountDeletionService | U1 |

**Validation**: 全 25 コンポーネント + 6 サービスが 1 以上の Unit に割り当てられている ✅
