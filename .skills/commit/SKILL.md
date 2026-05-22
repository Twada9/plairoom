---
name: commit
description: >
  このプロジェクト専用のgitコミットスキル。コミットを作成・実行するよう頼まれたとき、
  またはユーザーが「コミットして」「変更を保存して」「commitして」と言ったときに必ず使用すること。
  日本語コミットメッセージ、機能単位でのステージング、追加/修正の動詞ルールを自動で適用する。
---

# コミットスキル

このプロジェクトのgitコミット規約を適用してコミットを作成する。

---

## メッセージルール

### 動詞の選び方

**新しいファイルが含まれる、または機能が新規追加される →「〜追加」**
```
設定画面のゲスト表示を追加
SettingsFeature に isAuthenticated を追加
```

**既存ファイルの修正のみ（新規ファイルなし）→ 「〜修正」**
```
AppFeature の @Presents 対応を修正
ログアウト後の認証状態リセットを修正
```

### 禁止事項

- `feat:` `fix:` `chore:` などの conventional commit プレフィックス → 不要
- `ios` `android` などのプラットフォーム名 → 不要
- 英語メッセージ → 日本語で書く

---

## ステージング方針

変更を**機能のまとまり**ごとにグループ分けしてコミットを分ける。
1つの操作で複数の機能を変更した場合は複数コミットに分割する。

**グループ分けの例：**
- 同じ Feature（Reducer + View）は1コミット
- 複数 Feature にまたがる共通修正（AppFeature の修正など）は別コミット
- 設定ファイル・ドキュメントは機能コミットと分けてよい

1機能しか変更していない場合は1コミットでよい。

---

## 実行手順

### 1. 変更の把握

```bash
git status
git diff HEAD
```

新規ファイル（Untracked / new file）と変更ファイル（modified）を区別して確認する。

### 2. グループ分けとメッセージ決定

変更ファイルを機能ごとにグループ化する。各グループについて：
- 新規ファイルが1つでも含まれる → 「〜追加」
- 修正のみ → 「〜修正」

メッセージは体言止め（「〜した」ではなく「〜追加」「〜修正」）。

### 3. コミット実行

グループごとに以下を繰り返す：

```bash
# 対象ファイルを個別に指定してステージング
git add path/to/file1 path/to/file2

# HEREDOCでコミット
git commit -m "$(cat <<'EOF'
コミットメッセージ（日本語）

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

`git add .` や `git add -A` は使わず、ファイルを個別に指定する。

### 4. 確認

```bash
git log --oneline -5
```

コミットが正しく作成されたことを確認してユーザーに報告する。

---

## メッセージ例

| 変更内容 | メッセージ |
|---|---|
| SettingsView.swift を新規作成 | 設定画面を追加 |
| SettingsFeature.swift + SettingsView.swift 両方を新規作成 | 設定機能を追加 |
| ContentView.swift の @Presents 対応 | AppFeature の @Presents 対応を修正 |
| RoomListFeature + RoomDetailFeature を新規作成 | ルーム一覧・詳細機能を追加 |
| TCA スキルファイルを追加 | TCA 開発スキルを追加 |
| 複数 Feature の body 型を修正 | Reducer の body 返り値型を修正 |
