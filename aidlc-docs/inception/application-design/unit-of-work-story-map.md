# Unit of Work — Story Map

**Version**: 2.0
**Last Updated**: 2026-05-08
**Purpose**: Story ↔ Unit ↔ Priority (MVP/Full) の完全マッピング

---

## 1. Unit × Story Matrix

| Story | Title | Priority | Primary Unit | Supporting Units |
|---|---|---|---|---|
| US-01-01 | ユーザー登録 | MVP | **U1** identity | U7 (登録 UI), U8 (Cognito infra) |
| US-01-02 | ログイン & セッション維持 | MVP | **U1** identity | U7 (ログイン UI), U8 (Authorizer) |
| US-01-03 | 初回ヒアリング (ダメな欲望プロファイル) | MVP | **U1** identity | U6 (Pydantic), U7 (ヒアリング UI) |
| US-01-04 | 定時設定 (ダメモード突入時刻) | MVP | **U1** identity | U7 (設定 UI) |
| US-01-05 | プロファイル編集 | Full | **U1** identity | U7 (編集 UI) |
| US-02-01 | 動的頻度スケジューラ (堕落ランプ) | MVP | **U2** reminder | U7 (SchedulerClient FE-07) |
| US-02-02 | コンテキスト収集 | MVP | **U7** frontend | U6 (PlatformAdapter interface) |
| US-02-03 | AI コンテンツ選定 (ダメな未来ジェネレータ / Bedrock) | MVP | **U2** reminder | U4 (family/pet image pick), U7 (表示) |
| US-02-04 | ダメな未来コンテンツ提示 UI | MVP | **U7** frontend | U2 (API) |
| US-02-05 | リアクション (もう堕ちる) | MVP | **U3** termination | U7 (リアクションボタン) |
| US-02-06 | D3 近隣飲食店 (ダミー) | MVP | **U2** reminder | U7 (表示) |
| US-02-07 | D2/D5 趣味・推し (ダミー) | MVP | **U2** reminder | U7 (表示) |
| US-02-08 | D6 電車・天気 (ダミー) | MVP | **U2** reminder | U7 (表示) |
| US-02-09 | D4 酒テロ (ダミー) | MVP | **U2** reminder | U7 (表示) |
| US-02-10 | 提示履歴閲覧 | Full | **U2** reminder | U7 (履歴 UI) |
| US-03-01 | 堕落ゲート発動 (定時オーバーレイ) | MVP | **U7** frontend | U3 (バックエンド記録) |
| US-03-02 | ダミー勤怠画面 + 堕落宣言ボタン | MVP | **U7** frontend | U3 (/terminations/clock-out) |
| US-03-03 | 早期堕ち | MVP | **U3** termination | U7 (UI 起動), U5 (audit) |
| US-03-04 | ダメモード突入メッセージ | Full | **U3** termination | U2 (BE-04 ContentSelection 再利用) |
| US-04-01 | 写真アップロード (D1 素材, Pre-signed URL) | MVP | **U4** photo | U6 (validation), U7 (アップ UI) |
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
| U2 reminder | 7 | 2 | 1 (US-04-02) | 9+1 |
| U3 termination | 2 | 2 | — | 4 |
| U4 photo | 1 | 2 | 1 (US-04-02) | 3+1 |
| U5 audit-observability | 0 | 1 | 2 (US-03-03, US-05-04) | 1+2 |
| U6 shared-library | 0 | 0 | 多数 (横断) | 0 |
| U7 frontend | 5 | 0 | 全 MVP Story 対応 | 5+多数 |
| U8 infrastructure | 1 | 1 | 多数 (横断) | 2 |
| **合計 Primary** | **20** | **10** | — | — |

**内訳**:
- MVP: US-01-01, 01-02, 01-03, 01-04, 02-01, 02-02, 02-03, 02-04, 02-05, 02-06, 02-07, 02-08, 02-09 (D4 酒), 03-01, 03-02, 03-03, 04-01, 04-02, 05-01, 05-03 = **20 Story**
- Full: US-01-05, 02-10, 03-04, 04-03, 04-04, 05-02, 05-04, 05-05 = **8 Story**
- **計 28 Story** ✅

---

## 3. Critical Path to Demo (MVP 2 週間)

ハッカソンデモで必須の **20 Story を通す最短パス**:

```
Phase 0 (U6 + U8 bootstrap) :
  └─ (全 Story の前提)

Phase 1 Parallel :
  ├─ U1 identity : US-01-01 → 01-02 → 01-03 (ダメな欲望プロファイル) → 01-04
  ├─ U2 reminder : US-02-01 (堕落ランプ) → 02-03 (ダメな未来ジェネレータ)
  │                (+ 02-06 D3, 02-07 D2/D5, 02-08 D6, 02-09 D4 酒)
  └─ U4 photo    : US-04-01 (D1 素材) → 04-02 (U2 と連携)

Phase 2 :
  └─ U3 termination : US-02-05 (もう堕ちる) → 03-03 (早期堕ち) → 03-01 (FE) → 03-02 (FE)

Phase 3 (U7 frontend) :
  ├─ シェル + 認証 UI (US-01-01, 01-02)
  ├─ ヒアリング UI (ダメな欲望プロファイル US-01-03, 01-04)
  ├─ コンテキスト収集 FE (US-02-02, 05-01)
  ├─ ダメな未来コンテンツ提示 UI (US-02-04)
  ├─ SchedulerClient (堕落ランプ US-02-01 FE)
  ├─ リアクションボタン (US-02-05 FE)
  ├─ 堕落ゲート + ダミー勤怠 (US-03-01, 03-02)
  └─ 写真アップロード UI (US-04-01)

Phase 4 (CP-3) :
  └─ E2E デモシナリオ貫通テスト
```

---

## 4. Demo Scenario (審査員向け、約 3 分)

全 MVP Story を通す 1 本のデモシナリオ (冒頭にコンセプト解説を添える):

0. **(0:00)** コンセプト解説 (10 秒): 「このプロダクトは、仕事人間を "ダメな人間" に堕として、遊び・家族・推し・食事・酒・ペットといった "ダメな欲望" を充足させることで、明日の創造性を引き出します」
1. **(0:10)** 新規ユーザーとしてログイン。
2. **(0:40)** **ダメな欲望プロファイル** ヒアリング (10 問) に回答 (定時 18:00、猫 2 匹、サウナ好き、ラーメン好き、**ビール好き (D4)**、推しあり 等)。
3. **(1:10)** 時計を 17:00 に疑似設定。**堕落ランプ** 開始。**1 つ目のダメな未来** が登場 (「今夜のサウナは空いてる」— D5)。
4. **(1:30)** 時計 17:30、**近所のラーメン画像** が煽り文言付きで出てくる (D3)。
5. **(1:50)** 時計 17:50、**キンキンのビール画像 (D4 酒テロ)** が登場 — **「18:15 からビールが冷えてる」**
6. **(2:05)** 時計 17:55、**猫の写真** + 「ミケが玄関で待ってる」AI キャプション (D1)。
7. **(2:20)** 「もう堕ちる」ボタンを押す。**堕落ゲート** が全画面スライドイン + SE。コピー: **「今日の仕事は終わりだ、明日の自分に任せてダメになれ」**
8. **(2:30)** ダミー勤怠画面の **堕落宣言ボタン** クリック。ダメモード突入完了メッセージ: 「今日もおつかれ。さあ、明日の自分に任せよう」
9. **(2:45)** (Full 予告) 管理画面で履歴・プロファイル表示。
10. **(2:55)** 締め: 「働き手は明日、より良いアウトプットを出す。今日ダメになる自由が、明日の創造性です」

---

## 5. Validation Checklist

- [x] 全 28 Story が 1 以上の Unit に割り当てられている ✅
- [x] 全 25 Component (FE 9 + BE 12 + Cross 4) がいずれかの Unit に所属 ✅
- [x] 全 6 Service がいずれかの Unit に所属 ✅
- [x] MVP Story 20 の Primary Unit が U1〜U4, U7, U8 のいずれか (U5 は Full のみ、整合) ✅
- [x] 循環依存なし ✅
- [x] Critical Path (デモフロー) が Phase 0〜3 で完結する ✅
- [x] コンセプト用語 (ダメな欲望プロファイル / 堕落ランプ / 堕落ゲート / ダメな未来ジェネレータ) が Story / Unit / デモシナリオ全体で一貫 ✅
