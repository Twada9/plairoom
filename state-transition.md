# STATE TRANSITION DIAGRAM
**AI Content SNS — 状態遷移図 / FSM Specification · 2026-03-14**

---

## 01 — アプリ起動

> App 起動時の認証状態チェックフロー（AppFeature / Application）

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#1e2235', 'primaryBorderColor': '#00e5ff', 'lineColor': '#00e5ff', 'edgeLabelBackground': '#161616', 'stateBkg': '#1e2235', 'stateLabelColor': '#e0e0e0', 'transitionColor': '#00e5ff', 'fontFamily': 'monospace'}}}%%
stateDiagram-v2
    direction LR
    [*] --> Launching
    Launching --> AuthChecking
    AuthChecking --> LoggedIn     : 認証済み
    AuthChecking --> LoggedOut    : 未ログイン
    AuthChecking --> AuthRetrying : ロード失敗
    AuthRetrying --> AuthChecking : リトライ（1・2回目）
    AuthRetrying --> LoggedOut    : 3回失敗（未ログイン扱い）
    LoggedIn  --> [*] : ホーム画面へ
    LoggedOut --> [*] : ログインモーダル表示
```

---

## 02 — 全体：認証状態（Global Auth）

> アプリ全体で保持するセッション状態

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#1e2235', 'primaryBorderColor': '#00e5ff', 'lineColor': '#00e5ff', 'edgeLabelBackground': '#161616', 'stateBkg': '#1e2235', 'stateLabelColor': '#e0e0e0', 'transitionColor': '#00e5ff', 'fontFamily': 'monospace'}}}%%
stateDiagram-v2
    direction LR
    [*] --> Unauthenticated
    Unauthenticated --> Authenticated   : ログイン成功
    Authenticated   --> Unauthenticated : ログアウト
```

---

## 03 — 全体：コンテンツ生成状態（Global Generation）

> Authenticated と**並行して保持**。Supabase Realtime で `status` 変化を監視

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#1a2e1a', 'primaryBorderColor': '#4ade80', 'lineColor': '#4ade80', 'edgeLabelBackground': '#161616', 'stateBkg': '#1a2e1a', 'stateLabelColor': '#e0e0e0', 'transitionColor': '#4ade80', 'fontFamily': 'monospace'}}}%%
stateDiagram-v2
    direction LR
    [*] --> notGenerating
    notGenerating --> generating  : 生成リクエスト送信成功
    generating --> notGenerating  : 生成完了（Realtime 通知）
    generating --> notGenerating  : 生成失敗（Realtime or タイムアウト）
```

| 項目 | 内容 |
|---|---|
| 監視テーブル | `image_contents` / `music_contents` |
| フィルタ | `user_id = eq.{currentUserId}` |
| イベント | `UPDATE` — `status` が `generating → completed` に変化したタイミング |

---

## 04 — ルームリスト

> `RoomListFeature` (iOS · TCA) · `RoomListViewModel` (Android · MVVM)
>
> ⚑ Global Generation（03）と**並行して動作**

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#1e2235', 'primaryBorderColor': '#00e5ff', 'lineColor': '#00e5ff', 'edgeLabelBackground': '#161616', 'stateBkg': '#1e2235', 'stateLabelColor': '#e0e0e0', 'transitionColor': '#00e5ff', 'fontFamily': 'monospace'}}}%%
stateDiagram-v2
    [*] --> loading
    loading --> idle       : ロード完了
    loading --> loadFailed : ロード失敗
    loadFailed --> loading : リトライ
    loadFailed --> error   : リトライ失敗
    error --> loading : 再試行（ロード）を選択
    error --> idle    : キャンセルを選択
    idle --> requesting : ユーザーアクション
    requesting --> idle : 通信成功
    requesting --> idle : 通信失敗 [AppError]
    idle --> requesting : 手動リトライ
```

### AppError — エラー種別定義

| ErrorType | 説明 |
|---|---|
| `networkError` | ネットワーク障害 / タイムアウト |
| `usageLimitExceeded` | 無料プランの生成回数上限超過（サーバー側で判定・返却） |
| `serverError(code: Int)` | サーバーエラー（5xx 系） |
| `unauthorized` | 認証切れ → ログインモーダルへ誘導 |

---

## 05 — ルーム詳細

> `RoomDetailFeature` (iOS · TCA) · `RoomDetailViewModel` (Android · MVVM)
>
> ⚑ Global Generation（03）と**並行して動作**

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#1e2235', 'primaryBorderColor': '#00e5ff', 'lineColor': '#00e5ff', 'edgeLabelBackground': '#161616', 'stateBkg': '#1e2235', 'stateLabelColor': '#e0e0e0', 'transitionColor': '#00e5ff', 'fontFamily': 'monospace'}}}%%
stateDiagram-v2
    [*] --> loading
    loading --> idle       : ロード完了
    loading --> loadFailed : ロード失敗
    loadFailed --> loading : リトライ
    loadFailed --> error   : リトライ失敗
    error --> loading : 再試行（ロード）を選択
    error --> idle    : キャンセルを選択
    idle --> requesting : ユーザーアクション
    requesting --> idle : 通信成功
    requesting --> idle : 通信失敗 [AppError]
    idle --> requesting : 手動リトライ
    idle --> loginModal : 未ログインユーザーが生成ボタンをタップ
    loginModal --> idle  : ログイン成功 / キャンセル
```

---

## 06 — AI生成画面

> `GenerateFeature` (iOS · TCA) · `GenerateViewModel` (Android · MVVM)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#2e1a0a', 'primaryBorderColor': '#ff6b35', 'lineColor': '#ff6b35', 'edgeLabelBackground': '#161616', 'stateBkg': '#2e1a0a', 'stateLabelColor': '#e0e0e0', 'transitionColor': '#ff6b35', 'fontFamily': 'monospace'}}}%%
stateDiagram-v2
    direction LR
    [*] --> idle
    idle --> requesting : 送信ボタンタップ（Edge Function 呼び出し）
    requesting --> idle : 通信失敗 [AppError]
    requesting --> [*]  : 通信成功 → ルーム詳細へ戻る
```

**通信成功後の副作用**
- `image_contents.status` = `"generating"` で DB レコード作成済み（Edge Function が更新）
- Global Generation が `notGenerating → generating` に遷移
- `usageLimitExceeded` は `AppError` として返却 → `idle` に戻りエラー表示

---

## 07 — 生成結果画面

> `ResultFeature` (iOS · TCA) · `ResultViewModel` (Android · MVVM)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#2e1a0a', 'primaryBorderColor': '#ff6b35', 'lineColor': '#ff6b35', 'edgeLabelBackground': '#161616', 'stateBkg': '#2e1a0a', 'stateLabelColor': '#e0e0e0', 'transitionColor': '#ff6b35', 'fontFamily': 'monospace'}}}%%
stateDiagram-v2
    direction LR
    [*] --> idle
    idle --> requesting : 投稿ボタンタップ
    requesting --> idle : 通信失敗 [AppError]
    requesting --> [*]  : 通信成功 → ルーム詳細へ
    idle --> [*]        : キャンセル（戻るボタン）→ ルーム詳細へ
    idle --> [*]        : やり直す → AI生成画面へ
```

---

## 08 — ログイン / 新規登録モーダル

> `AuthFeature` (iOS · TCA) · `AuthViewModel` (Android · MVVM)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#2a1a3e', 'primaryBorderColor': '#a855f7', 'lineColor': '#a855f7', 'edgeLabelBackground': '#161616', 'stateBkg': '#2a1a3e', 'stateLabelColor': '#e0e0e0', 'transitionColor': '#a855f7', 'fontFamily': 'monospace'}}}%%
stateDiagram-v2
    direction LR
    [*] --> loginIdle
    loginIdle  --> signUpIdle  : 新規登録タブ切り替え
    signUpIdle --> loginIdle   : ログインタブ切り替え
    loginIdle  --> requesting  : ログインボタンタップ
    signUpIdle --> requesting  : 登録ボタンタップ
    requesting --> loginIdle   : 通信失敗 [AppError]
    requesting --> [*]         : 通信成功 → モーダルを閉じる
    loginIdle  --> [*]         : キャンセル
    signUpIdle --> [*]         : キャンセル
```

---

## 09 — 設定画面

> `SettingsFeature` (iOS · TCA) · `SettingsViewModel` (Android · MVVM)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#1e2235', 'primaryBorderColor': '#00e5ff', 'lineColor': '#00e5ff', 'edgeLabelBackground': '#161616', 'stateBkg': '#1e2235', 'stateLabelColor': '#e0e0e0', 'transitionColor': '#00e5ff', 'fontFamily': 'monospace'}}}%%
stateDiagram-v2
    direction LR
    [*] --> idle
    idle --> [*] : ログアウト → Global Auth が Unauthenticated へ遷移
```
