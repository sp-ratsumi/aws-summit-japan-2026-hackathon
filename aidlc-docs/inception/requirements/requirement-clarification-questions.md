# Requirements Clarification Questions — アフターファイブ

最初の回答を分析したところ、いくつか整合性の確認が必要な点がありましたので、4 問だけ追加で確認させてください。

---

## Clarification 1: Tauri デスクトップアプリ と フロントエンド配信の関係

**検出された整合性の論点**:
- Q4 で「Tauri デスクトップアプリ」を選択 (フロントエンドはアプリにバンドル)
- Q19 で「CloudFront + S3 (フロント)」を含むサーバーレス構成を選択
- Q20 で「React + TypeScript + Vite」を選択

Tauri は React + Vite で作った SPA をネイティブアプリとして配信するため、通常はフロントエンドを CloudFront + S3 でホストしません。

### Clarification Question 1
フロントエンドの配信モデルはどれですか?

A) **Tauri アプリのみ** (CloudFront+S3 はフロント配信には使わず、アップロード画像の配信のみ)
B) **両対応** (Tauri アプリ版 + Web 版、両方をデモで見せたい) — 同じ React コードから tauri build と web build の 2 系統を生成
C) **Tauri アプリのみ、ただし配布インストーラー/アップデータ配信に CloudFront+S3 を使う**
D) **Web 版のみ** (Q4 の Tauri 回答を撤回し、PWA にする)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Clarification 2: 通知の実装方式 (Tauri での「Web Push」)

**検出された整合性の論点**:
- Q4 で Tauri デスクトップアプリを選択
- Q22 で「Web Push 通知 (ブラウザ)」を選択

Tauri デスクトップアプリは一般的に OS ネイティブ通知 (`tauri-plugin-notification`) を使います。ブラウザの Web Push API はデスクトップアプリと相性が良くありません。

### Clarification Question 2
通知の実装方式はどれがよいですか?

A) **OS ネイティブ通知** (`tauri-plugin-notification` で macOS / Windows の通知センターに出す) — Tauri と相性良く、MVP で現実的
B) **アプリ内通知のみ** (アプリ起動中のモーダル/トーストだけ)
C) **両方**: アプリ起動中はアプリ内通知、バックグラウンド時は OS ネイティブ通知
D) **バックエンドから SNS/EventBridge 経由でプッシュし、Tauri 側で受けて OS 通知**
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Clarification 3: 「認証なし」と Security Baseline の両立

**検出された整合性の論点**:
- Q5 で「認証不要」を選択
- Q23 で「S3 Pre-signed URL で直接アップロード」を選択
- 拡張機能で **Security Baseline 有効化** を選択 (SECURITY-08: Application-Level Access Control、SECURITY-12: Authentication and Credential Management)

認証がないと、アップロード画像の所有者を判定できず、他ユーザーの画像を上書き/閲覧できる IDOR リスクが生じます。Security Baseline の blocking finding になります。

### Clarification Question 3
どのように両立させますか?

A) **Cognito Identity Pool (未認証 ID)** を使い、匿名でもユーザー識別子 (IdentityId) を持たせる — 「ログインなし」の UX は維持しつつ、S3/API 側は Identity ベースで所有権チェック可能
B) **フロント生成の device ID (UUID)** を LocalStorage に保存し、それをリソースキーに使う — 最もシンプルだが IDOR 対策は弱い
C) **シンプルな Cognito User Pool 認証 (メール + パスワード)** に切り替える (Q5 を再考)
D) **デモ専用アプリと割り切り、single-demo-user 前提で全員が共有** する (ユーザー分離なし)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Clarification 4: ヒアリング結果の DB 永続化

**検出された整合性の論点**:
- Q16 で「A: AI (Bedrock) プロンプトに含める」を選択 (選択肢 C:「DB + AI 両方」は不選択)
- Q24 で「C: プロファイル + 画像 + 提示履歴 + リアクション まで永続化」を選択

Q24 がプロファイルの永続化を含む一方、Q16 は DB 保存を選ばれていません。論理的には「DB に保存 + AI プロンプトにも使う」が整合的 (Q16=C) ですが、確認させてください。

### Clarification Question 4
ヒアリング結果はどう扱いますか?

A) **DB (DynamoDB) に保存し、かつ AI プロンプトに毎回渡す** (Q16 を C 相当に読み替える、整合性あり)
B) **DB に保存しない (セッション内メモリまたはクライアント LocalStorage のみ)、AI プロンプトにだけ使う** (Q24 からはプロファイルを除外、Q24=B 相当)
C) **DB に保存するが AI は使わない (Q16=B 相当)**
X) Other (please describe after [Answer]: tag below)

[Answer]: A
