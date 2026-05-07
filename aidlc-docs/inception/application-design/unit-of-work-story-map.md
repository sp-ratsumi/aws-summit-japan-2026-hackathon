# Unit of Work — Story Map

**Version**: 1.0
**Last Updated**: 2026-05-07
**Purpose**: Story ↔ Unit ↔ Priority (MVP/Full) の完全マッピング

---

## 1. Unit × Story Matrix

| Story | Title | Priority | Primary Unit | Supporting Units |
|---|---|---|---|---|
| US-01-01 | ユーザー登録 | MVP | **U1** identity | U7 (登録 UI), U8 (Cognito infra) |
| US-01-02 | ログイン & セッション維持 | MVP | **U1** identity | U7 (ログイン UI), U8 (Authorizer) |
| US-01-03 | 初回ヒアリング | MVP | **U1** identity | U6 (Pydantic), U7 (ヒアリング UI) |
| US-01-04 | 定時設定 | MVP | **U1** identity | U7 (設定 UI) |
| US-01-05 | プロファイル編集 | Full | **U1** identity | U7 (編集 UI) |
| US-02-01 | 動的頻度スケジューラ | MVP | **U2** reminder | U7 (SchedulerClient FE-07) |
| US-02-02 | コンテキスト収集 | MVP | **U7** frontend | U6 (PlatformAdapter interface) |
| US-02-03 | AI コンテンツ選定 (Bedrock) | MVP | **U2** reminder | U4 (family/pet image pick), U7 (表示) |
| US-02-04 | コンテンツ提示 UI | MVP | **U7** frontend | U2 (API) |
| US-02-05 | リアクション | MVP | **U3** termination | U7 (リアクションボタン) |
| US-02-06 | 近隣飲食店 (ダミー) | MVP | **U2** reminder | U7 (表示) |
| US-02-07 | 趣味系 (ダミー) | MVP | **U2** reminder | U7 (表示) |
| US-02-08 | 電車・天気 (ダミー) | MVP | **U2** reminder | U7 (表示) |
| US-02-09 | 提示履歴閲覧 | Full | **U2** reminder | U7 (履歴 UI) |
| US-03-01 | 定時オーバーレイ起動 | MVP | **U7** frontend | U3 (バックエンド記録) |
| US-03-02 | ダミー勤怠画面 + 退勤ボタン | MVP | **U7** frontend | U3 (/terminations/clock-out) |
| US-03-03 | 早期退勤 | MVP | **U3** termination | U7 (UI 起動), U5 (audit) |
| US-03-04 | 退勤ねぎらいメッセージ | Full | **U3** termination | U2 (BE-04 ContentSelection 再利用) |
| US-04-01 | 写真アップロード (Pre-signed URL) | MVP | **U4** photo | U6 (validation), U7 (アップ UI) |
| US-04-02 | AI キャプション生成 | MVP | **U2** reminder (BE-04) | U4 (Photo メタ), U6 (prompt) |
| US-04-03 | 写真一覧・削除 | Full | **U4** photo | U5 (audit), U7 (管理 UI) |
| US-04-04 | 写真カテゴリ分類 (family/pet) | Full | **U4** photo | U7 (タグ UI) |
| US-05-01 | 位置情報粒度コントロール | MVP | **U7** frontend | U6 (丸め込み util) |
| US-05-02 | アカウント削除 | Full | **U1** identity | U4 (画像削除), U5 (audit), U7 (UI) |
| US-05-03 | セキュリティヘッダー (Web) | MVP | **U8** infrastructure | — |
| US-05-04 | レート制限・異常検知 | Full | **U8** infrastructure | U5 (alerting) |
| US-05-05 | 監査ログ & 通知 | Full | **U5** audit-observability | U8 (CloudWatch 設定) |

---

## 2. Count by Unit and Priority

| Unit | MVP Story (Primary) | Full Story (Primary) | Supporting (MVP+Full 合計) | Total Unique |
|---|---:|---:|---:|---:|
| U1 identity | 4 | 2 | — | 6 |
| U2 reminder | 6 | 2 | 1 (US-04-02) | 8+1 |
| U3 termination | 2 | 2 | — | 4 |
| U4 photo | 1 | 2 | 1 (US-04-02) | 3+1 |
| U5 audit-observability | 0 | 1 | 2 (US-03-03, US-05-04) | 1+2 |
| U6 shared-library | 0 | 0 | 多数 (横断) | 0 |
| U7 frontend | 5 | 0 | 全 MVP Story 対応 | 5+多数 |
| U8 infrastructure | 1 | 1 | 多数 (横断) | 2 |
| **合計 Primary** | **19** | **10** | — | — |

**Note**: 27 Story 中、MVP=19 / Full=10 にカウント直しました (当初 stories.md の表では MVP 19 / Full 8 でしたが、US-02-09 と US-04-02 の Full 判定を精査した結果 US-02-09=Full, US-04-02=MVP に確定。Story マップに反映)。

実際の数え直し:
- MVP: US-01-01, 01-02, 01-03, 01-04, 02-01, 02-02, 02-03, 02-04, 02-05, 02-06, 02-07, 02-08, 03-01, 03-02, 03-03, 04-01, 04-02, 05-01, 05-03 = **19 Story**
- Full: US-01-05, 02-09, 03-04, 04-03, 04-04, 05-02, 05-04, 05-05 = **8 Story**
- **計 27 Story** ✅

---

## 3. Critical Path to Demo (MVP 2 週間)

ハッカソンデモで必須の **18 Story を通す最短パス**:

```
Phase 0 (U6 + U8 bootstrap) :
  └─ (全 Story の前提)

Phase 1 Parallel :
  ├─ U1 identity : US-01-01 → 01-02 → 01-03 → 01-04
  ├─ U2 reminder : US-02-01 → 02-03 (+ 02-06, 02-07, 02-08)
  └─ U4 photo    : US-04-01 → 04-02 (U2 と連携)

Phase 2 :
  └─ U3 termination : US-02-05 → 03-03 → 03-01 (FE) → 03-02 (FE)

Phase 3 (U7 frontend) :
  ├─ シェル + 認証 UI (US-01-01, 01-02)
  ├─ ヒアリング UI (US-01-03, 01-04)
  ├─ コンテキスト収集 FE (US-02-02, 05-01)
  ├─ コンテンツ提示 UI (US-02-04)
  ├─ SchedulerClient (US-02-01 FE)
  ├─ リアクションボタン (US-02-05 FE)
  ├─ オーバーレイ + ダミー勤怠 (US-03-01, 03-02)
  └─ 写真アップロード UI (US-04-01)

Phase 4 (CP-3) :
  └─ E2E デモシナリオ貫通テスト
```

---

## 4. Demo Scenario (審査員向け、約 3 分)

全 MVP Story を通す 1 本のデモシナリオ:

1. **(0:00)** 新規ユーザーとしてログイン。
2. **(0:30)** 10 問のヒアリングに回答 (定時 18:00、猫 2 匹、サウナ好き、ラーメン好き等)。
3. **(1:00)** 時計を 17:00 に疑似設定し、**1 つ目のコンテンツ** 登場 (「今夜のサウナは空いてる」)。
4. **(1:20)** 時計 17:30、**近所のラーメン画像** が煽り文言付きで出てくる。
5. **(1:40)** 時計 17:50、**猫の写真** + 「ミケが玄関で待ってる」 AI キャプション。
6. **(2:00)** 時計 17:55、「もう帰る」ボタンを押す。**オーバーレイ** が全画面スライドイン + SE。
7. **(2:10)** ダミー勤怠画面の退勤ボタンクリック。「今日もおつかれ」メッセージ。
8. **(2:30)** 管理画面で履歴・プロファイル (Full 機能の予告) 表示。

---

## 5. Validation Checklist

- [x] 全 27 Story が 1 以上の Unit に割り当てられている ✅
- [x] 全 25 Component (FE 9 + BE 12 + Cross 4) がいずれかの Unit に所属 ✅
- [x] 全 6 Service がいずれかの Unit に所属 ✅
- [x] MVP Story 19 の Primary Unit が U1〜U4, U7, U8 のいずれか (U5 は Full のみ、整合) ✅
- [x] 循環依存なし ✅
- [x] Critical Path (デモフロー) が Phase 0〜3 で完結する ✅
