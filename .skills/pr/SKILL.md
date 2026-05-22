---
name: pr
description: >
  このプロジェクト専用のPull Request作成スキル。「PRを作って」「プルリクを出して」
  「PR作成して」と言われたとき、またはブランチの変更をレビュー依頼したいときに必ず使用すること。
  ブランチの差分を分析し、日本語でタイトル・本文を生成してGitHub PRを作成する。
---

# PR作成スキル

現在のブランチとベースブランチの差分を分析し、日本語でPRを作成する。

---

## タイトルルール

コミットスキルと同じ動詞を使う。

**新規ファイル・機能追加が含まれる → 「〜追加」**
```
設定画面のゲスト/ログイン状態分岐を追加
```

**既存コードの修正のみ → 「〜修正」**
```
AppFeature の @Presents 対応を修正
```

**複数の変更が混在する → 変更の主旨を一言で表す**
```
設定画面の認証状態対応
```

- `feat:` などのプレフィックス不要
- `ios` `android` などのプラットフォーム名不要
- 50文字以内を目安に

---

## 本文フォーマット

```markdown
## 概要
（この PR で何をしたか、なぜ必要だったかを2〜3文で）

## 変更内容
- ファイル名 or 機能名：変更の説明
- ファイル名 or 機能名：変更の説明

## 動作確認
- [ ] 確認項目1
- [ ] 確認項目2
```

動作確認は変更内容から推測して記載する。自動テストがない場合は手動確認項目を入れる。

---

## 実行手順

### 1. 現在の状態を確認

```bash
# 現在のブランチ名
git branch --show-current

# ベースブランチを確認（main / develop / master など）
git remote show origin | grep "HEAD branch"

# このブランチのコミット一覧
git log main..HEAD --oneline
# または
git log develop..HEAD --oneline

# 差分の概要（変更ファイル一覧）
git diff main..HEAD --name-status
# または
git diff develop..HEAD --name-status
```

### 2. 差分の内容を把握

```bash
# 変更内容の詳細（必要に応じて）
git diff main..HEAD
```

コミットメッセージと変更ファイルから、PRのタイトルと本文を決定する。

### 3. PR を作成

未pushの場合は先にpushする：

```bash
git push -u origin HEAD
```

PR作成：

```bash
gh pr create \
  --title "タイトル（日本語）" \
  --body "$(cat <<'EOF'
## 概要
〜

## 変更内容
- 〜

## 動作確認
- [ ] 〜
EOF
)"
```

### 4. 作成確認

```bash
gh pr view --web
```

PR の URL をユーザーに伝える。

---

## ベースブランチの選び方

| 状況 | ベースブランチ |
|---|---|
| `feature/*` ブランチ | `main` または `develop` |
| `develop` ブランチ | `main` |
| 明示された場合 | 指定に従う |

`gh pr create` はデフォルトでリポジトリのデフォルトブランチをベースにするので、
特に指定がない場合は `--base` を省略してよい。
