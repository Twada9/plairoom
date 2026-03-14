# 要件定義書

## アプリ概要

AIを使ってコンテンツ（画像・音楽）を生成し、互いに評価し合って遊ぶSNSアプリ。
ユーザーはお題が設定された「部屋」にアクセスし、AIでコンテンツを生成・投稿・評価する。

---

## ユーザー種別

| 種別 | 説明 |
|---|---|
| ゲスト（未登録） | 閲覧のみ可能 |
| 登録ユーザー（無料） | AI生成に回数制限あり、いいね・コメント可能 |
| 登録ユーザー（プレミアム） | AI生成無制限、いいね・コメント可能 |

---

## 機能要件

### 認証
- メールアドレス・パスワードによるログイン・新規登録
- 未登録ユーザーがいいね・コメント・AI生成を操作した際にログインモーダルを表示
- ログアウト

### 部屋（Room）
- 部屋一覧の表示
- 部屋の作成・編集・削除は管理者のみ（Supabaseダッシュボードから操作）
- 部屋にはコンテンツタイプ（画像 or 音楽）が1種類決まっている
- 部屋にはルームタイプ（free / battle / quiz / collaboration）がある
- 誰でも閲覧可能

### コンテンツ生成
- AIを使って画像または音楽を生成する
- 部屋のお題（base_prompt）が自動で付加される
- ユーザーが自由にプロンプトを追記できる
- 生成はバックグラウンドで処理し、完了後に通知またはバナーで知らせる
- 生成完了後にユーザーが確認画面で投稿するかを決める
- 生成中・確認待ち・投稿済み・失敗のステータスを管理する
- 無料ユーザーは生成回数に上限あり（上限数は未定）
- プレミアムユーザーは生成無制限

### コンテンツ投稿・閲覧
- 生成したコンテンツをルームに投稿できる
- 投稿済みコンテンツは全員が閲覧可能
- 生成中・確認待ちのコンテンツは本人のみ閲覧可能
- ルーム詳細で自分のコンテンツにはステータスバッジを表示
- コンテンツの編集は不可
- コンテンツの削除は本人のみ可能

### いいね
- 登録ユーザーのみ可能
- 1コンテンツに1ユーザー1回のみ
- 取り消し可能

### コメント
- 登録ユーザーのみ可能
- 自分のコメントのみ削除可能

### 設定
- アカウント情報の確認・編集（名前・アバター）
- プラン状況の確認
- 課金ページへの導線
- プライバシーポリシー
- 利用規約
- ログアウト

---

## 非機能要件

- iOS / Android 両対応
- ダークモード対応
- オフライン時の挙動は現時点では未定

---

## 画面一覧

| 画面名 | 遷移方法 | 説明 |
|---|---|---|
| ルームリスト | 初期表示（Tab1） | 部屋の一覧 |
| ルーム詳細 | プッシュ遷移 | お題・コンテンツ一覧・生成ボタン |
| AI生成画面 | プッシュ遷移 | プロンプト入力・送信 |
| 生成結果画面 | プッシュ遷移 | 結果確認・投稿・やり直し |
| 設定画面 | Tab2 | アカウント情報・課金・ログアウト |
| ログイン画面 | モーダル | ログイン・新規登録（同画面内切り替え） |

---

## データベース設計

### テーブル一覧

#### profiles
| カラム | 型 | 説明 |
|---|---|---|
| id | uuid | Supabase Auth の user_id と紐付け |
| name | text | ユーザー名 |
| avatar_url | text | アバター画像URL |
| is_premium | boolean | プレミアムプランかどうか |
| created_at | timestamptz | 作成日時 |

#### rooms
| カラム | 型 | 説明 |
|---|---|---|
| id | uuid | |
| title | text | 部屋名 |
| description | text | 説明文 |
| base_prompt | text | お題・AIへの基本プロンプト |
| room_type | text | free / battle / quiz / collaboration |
| content_type | text | image / music |
| created_at | timestamptz | |

#### image_contents
| カラム | 型 | 説明 |
|---|---|---|
| id | uuid | |
| user_id | uuid | profiles への参照 |
| room_id | uuid | rooms への参照 |
| file_url | text | Supabase Storage の URL |
| prompt_used | text | 実際に使ったプロンプト |
| status | text | generating / pending / completed / failed |
| created_at | timestamptz | |

#### music_contents
| カラム | 型 | 説明 |
|---|---|---|
| id | uuid | |
| user_id | uuid | profiles への参照 |
| room_id | uuid | rooms への参照 |
| file_url | text | Supabase Storage の URL |
| prompt_used | text | 実際に使ったプロンプト |
| duration | integer | 再生時間（秒） |
| status | text | generating / pending / completed / failed |
| created_at | timestamptz | |

#### likes
| カラム | 型 | 説明 |
|---|---|---|
| id | uuid | |
| user_id | uuid | profiles への参照 |
| content_type | text | image / music |
| content_id | uuid | 外部キー制約なし（アプリ側で管理） |
| created_at | timestamptz | |

#### comments
| カラム | 型 | 説明 |
|---|---|---|
| id | uuid | |
| user_id | uuid | profiles への参照 |
| content_type | text | image / music |
| content_id | uuid | 外部キー制約なし（アプリ側で管理） |
| body | text | コメント本文 |
| created_at | timestamptz | |

#### ai_usage_logs
| カラム | 型 | 説明 |
|---|---|---|
| id | uuid | |
| user_id | uuid | profiles への参照 |
| content_type | text | image / music |
| used_at | timestamptz | 生成日時 |

---

## RLS設計

| テーブル | ゲスト | 登録ユーザー | 管理者 |
|---|---|---|---|
| profiles | 名前・アバターのみ読み取り可 | 自分のみ全項目読み書き | 全て |
| rooms | 読み取り可 | 読み取り可 | 作成・編集・削除 |
| image_contents | completed のみ読み取り可 | 作成・削除（自分のみ） | 全て |
| music_contents | completed のみ読み取り可 | 作成・削除（自分のみ） | 全て |
| likes | 読み取り可 | 作成・削除（自分のみ） | 全て |
| comments | 読み取り可 | 作成・削除（自分のみ） | 全て |
| ai_usage_logs | 不可 | 自分のみ読み取り | 全て |

- 管理者操作はSupabaseダッシュボードから直接実施（service_role使用）
- ai_usage_logs の書き込みはEdge Functionのみ

---

## 未確定事項

- 無料ユーザーのAI生成回数上限（マネタイズ方針が決まり次第確定）
- オフライン時の挙動
- プッシュ通知の実装方法（生成完了通知）
- 音楽生成AIサービスの選定
- 画像生成AIサービスの選定（DALL-E、Stable Diffusion など）
- quiz / collaboration のルームタイプの詳細仕様
- テキストコンテンツ（AI生成限定）の追加は将来検討
