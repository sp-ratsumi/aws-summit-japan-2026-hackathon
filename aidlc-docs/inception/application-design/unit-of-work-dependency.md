# Unit of Work Dependency — アフターファイブ

**Version**: 1.0
**Last Updated**: 2026-05-07

---

## 1. Dependency Matrix

行: depends on → / 列: 依存対象 / 値: ◉=強依存 / ○=弱依存 (read-only API or API 契約のみ)

| ↓ Unit / Depends → | U1 | U2 | U3 | U4 | U5 | U6 | U7 | U8 |
|---|---|---|---|---|---|---|---|---|
| **U1 identity** |   |   |   |   |   | ◉ |   | ◉ |
| **U2 reminder** |   |   |   | ○ |   | ◉ |   | ◉ |
| **U3 termination** |   | ◉ |   |   |   | ◉ |   | ◉ |
| **U4 photo** |   |   |   |   |   | ◉ |   | ◉ |
| **U5 audit-observability** |   |   |   |   |   | ◉ |   | ◉ |
| **U6 shared-library** |   |   |   |   |   |   |   |   |
| **U7 frontend** | ○ | ○ | ○ | ○ |   | ◉ |   | ○ |
| **U8 infrastructure** |   |   |   |   |   | ◉ |   |   |

### 凡例

- **◉ 強依存**: 設計・コンパイル時にブロッキング依存 (API 型、Lambda Layer、CDK リソース共有)
- **○ 弱依存**: API 契約を通じた runtime 依存のみ (別 Unit が動作していなくても当該 Unit の開発は進行可能)

### 読み方の例

- **U2 reminder → U4 photo (○)**: U2 が family/pet 写真を提示するときに U4 の Photo メタを読み取るが、API 契約 (OpenAPI) が合意されていれば dummy で開発可能
- **U3 termination → U2 reminder (◉)**: U3 のねぎらい文生成が U2 の BE-04 を直接呼び出すため、U2 のコードが先行必要

### Cyclic Dependency Check

- 上記マトリクスは **完全に上三角 / 下三角**: ✅ **循環依存なし**

---

## 2. Build / Development Order (U5=A 依存順)

### Phase 0: Foundation (直列)

```
U6 shared-library  →  U8 infrastructure (bootstrap)
```

これらが他すべての Unit のブロッキング依存。まず直列で完成させる。

**Phase 0 の完了条件**:
- `packages/shared-py/` パッケージが build 可能、Layer として deploy 可能
- `openapi/schema.yaml` のスケルトンが定まり、Pydantic 生成スクリプトが回る
- `infrastructure/` が `cdk deploy` 可能、AuthStack + DataStack + 空 ApiStack が AWS に存在する
- Cognito User Pool / DynamoDB テーブル / S3 PhotoBucket が deploy 完了

### Phase 1: Core Backend Units (並列可 — U6=C 部分並列)

Phase 0 完了後、以下を **並列** で開発可能:

```
┌─ U1 identity ──┐
├─ U2 reminder ──┤    (U3 は U2 完了まで待つ)
└─ U4 photo ─────┘
```

**並列化の根拠**: U1, U2, U4 は互いに強依存なし (U2→U4 が○ だけ)。API 契約 (OpenAPI schema) が先に確定していれば互いの実装完了を待たずに進められる。

### Phase 2: Downstream Backend Units (部分並列)

```
U3 termination (U2 完了後)
U5 audit-observability (Phase 1 のどれかが完了し、EventBridge イベントが流れ始めた後)
```

U3 と U5 は並列でも可。

### Phase 3: Frontend (バックエンド API が stub で動けば並列可)

```
U7 frontend
```

API 契約 (OpenAPI) が Phase 0 で凍結されていれば、バックエンド完成を待たずに mock server (例: Prism) で開発着手できる。実環境 E2E テストだけ Phase 1-2 完了後にやる。

### Summary Gantt-Style View

```
Timeline:
  Day 0-1:  [U6 shared-library + U8 bootstrap]
  Day 2-4:  [U1 identity] [U2 reminder] [U4 photo]  ← 3並列
  Day 5-6:  [U3 termination]  [U5 audit]              ← 2並列
  Day 2-7:  [U7 frontend]     (Day 2 から並行、Day 7 で E2E)
  Day 8:    [Build and Test]  (Code Generation + 全 Unit 統合)
```

※ ハッカソン 2 週間 MVP 想定。Full 版追加機能は Day 9〜14 で増築。

---

## 3. Communication Between Units

### 3.1 Shared Library (U6) 経由の共有

- **Pydantic モデル**: OpenAPI → `openapi-generator` or `datamodel-code-generator` で生成 → `packages/shared-py/afterfive_shared/models/`
- **TypeScript 型**: OpenAPI → `openapi-typescript` で生成 → `packages/shared-ts/src/generated/`
- **Repository クラス**: `packages/shared-py/afterfive_shared/repositories/` に配置、全 Unit で import

### 3.2 API 経由の runtime 通信

- **U3 → U2**: `ContentSelectionComponent.generate_closing_message()` を shared-py 経由で同一 Lambda 内 import 呼び出し (U9=B Route 1:1 の場合は U3 用 Lambda に BE-04 のコードを shared layer で載せる)
  - または EventBridge 経由に変更可 (将来的な疎結合化オプション)

### 3.3 EventBridge (非同期イベント) 経由

- Producer: U1 (onboarded, profile.updated, account.deletion.*), U3 (reaction.leave_now_requested, user.terminated), U4 (photo.uploaded, photo.deleted)
- Consumer: U5 (audit-observability) が全 event を subscribe
- Event Bus: `afterfive-bus` (U8 EventBusStack で定義)

### 3.4 DynamoDB 共有テーブル

- 全 Unit が Single-Table (`AfterFiveTable`) を共有
- **Partition Key 規約** でデータ分離 (例: `USER#<userId>` prefix は U1/U2/U3/U4 が触るが、SK prefix で Entity を区別)
- U5 は `AuditTable` を専有 (append-only)

---

## 4. Parallel Development Protocol (部分並列 — U6=C)

### 4.1 API Contract Lock (先行必要)

Phase 0 完了時点で `openapi/schema.yaml` に **MVP 範囲の全エンドポイント** を凍結 (Lock)。以降、各 Unit は schema を変更せずに実装可能。変更が必要な場合は:

1. Issue (PR) を上げて schema を先に更新
2. 全 Unit のメンテナに通知 (`docs/CHANGELOG.schema.md` に記録)
3. 型生成を再実行、影響 Unit でのコンパイルエラー解消

### 4.2 Mock Server for Frontend

フロントが Phase 1 の backend 完成を待たず開発するために:

- `openapi` に stoplight/prism または msw で mock server を立てる
- フロントは `VITE_API_BASE_URL` を mock / staging / prod で切替

### 4.3 Integration Checkpoints

Phase 完了ごとに統合テスト checkpoint を設ける:

| Checkpoint | 対象 | 検証 |
|---|---|---|
| **CP-0** | Phase 0 完了 | Layer build + CDK deploy + Cognito signup smoke test |
| **CP-1** | Phase 1 完了 | U1+U2+U4 API が OpenAPI 通り動く、PBT 単体テスト 全 pass |
| **CP-2** | Phase 2 完了 | 全 Backend Unit 動作、EventBridge → Audit 到達 |
| **CP-3** | Phase 3 完了 | Tauri + Web 両方でコアシナリオ (17:00 → 18:00) E2E pass |

---

## 5. Rollback Strategy

MVP 段階で本番データを持たない前提:

| 失敗シナリオ | 対応 |
|---|---|
| U6 の Layer build 失敗 | 上流 Unit のローカル開発は可、CI だけブロック。Layer を手元 wheel でも良い |
| U8 CDK deploy 失敗 | `cdk destroy` → 修正 → 再 deploy。Cognito / DynamoDB はテストデータのみなので delete OK |
| U1 認証フロー失敗 | Cognito Pre-Signup trigger を一旦削除 → 手動ユーザー作成でテスト続行 |
| U2 Bedrock 失敗 | フォールバック (テンプレート) で常時動作、API キー / IAM 権限を確認 |
| U3 EventBridge 失敗 | Lambda 内で直接 AuditComponent 関数を呼ぶ緊急 fallback を用意 |
| U7 Tauri build 失敗 | Web 版単独デモに切替 (同一コードベースなので可) |

---

## 6. Risk Register (Unit Dependency 観点)

| Risk | Unit | Mitigation |
|---|---|---|
| `shared-py` の API ブレークが全 Unit に波及 | U6 | Layer バージョン管理、breaking change は明示 PR |
| OpenAPI の不整合で型生成失敗 | U6 | CI で schema-lint + 生成コード diff を PR 必須 |
| `AfterFiveTable` の PK/SK 設計ミスで Unit 間衝突 | U1〜U5 | U6 Repository でパス命名を一元化、実装は Repository 経由のみ許可 |
| Lambda Authorizer の設定漏れで全エンドポイントが 403 | U8, U1 | Smoke test (CP-0) で必ず JWT 経由の 200 を確認 |
| Bedrock 呼び出しで PII 流出 | U2 | U2 の prompt builder で PII redaction をテストする (SEC-03) |

---

## 7. Content Validation

- [x] Mermaid を使用していない (マトリクス と Gantt はテキスト表現)
- [x] 依存マトリクスで循環依存をチェック → なし
- [x] 全 8 Unit が登場、重複なし
