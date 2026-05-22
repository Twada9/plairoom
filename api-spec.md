# API 仕様書
**AI Content SNS — API Specification · 2026-03-14**

---

## 共通仕様

### Base URL

| 種別 | URL |
|---|---|
| Supabase REST API | `https://{PROJECT_REF}.supabase.co/rest/v1` |
| Supabase Auth API | `https://{PROJECT_REF}.supabase.co/auth/v1` |
| Edge Functions | `https://{PROJECT_REF}.supabase.co/functions/v1` |

### 共通リクエストヘッダー

| ヘッダー | 値 | 必須 |
|---|---|---|
| `apikey` | `{SUPABASE_ANON_KEY}` | ✅ 全リクエスト |
| `Authorization` | `Bearer {JWT_TOKEN}` | ✅ 認証が必要なエンドポイント |
| `Content-Type` | `application/json` | POST / PATCH |
| `Prefer` | `return=representation` | レスポンスにレコードを含めたい場合 |

### エラーレスポンス形式

**Supabase REST / Auth エラー**
```json
{
  "code": "PGRST116",
  "details": "...",
  "hint": null,
  "message": "エラーの説明"
}
```

**Edge Functions カスタムエラー**
```json
{
  "error": {
    "type": "usageLimitExceeded",
    "message": "月の生成回数上限に達しました"
  }
}
```

**Edge Functions エラー種別**

| type | HTTP | 説明 |
|---|---|---|
| `usageLimitExceeded` | 429 | 無料プランの生成回数上限超過 |
| `unauthorized` | 401 | 未ログイン / JWT 期限切れ |
| `invalidCredentials` | 401 | 再認証失敗（退会時のパスワード不一致など） |
| `serverError` | 500 | AI サービスまたはサーバー内部エラー |
| `networkError` | 503 | AI サービスへの接続失敗 |

---

## 1. Auth API

### 1-1. 新規登録

```
POST /auth/v1/signup
```

**Request Body**
```json
{
  "email": "user@example.com",
  "password": "password123",
  "options": {
    "data": {
      "name": "ユーザー名"
    }
  }
}
```

**Response `200`**
```json
{
  "access_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 3600,
  "refresh_token": "...",
  "user": {
    "id": "uuid",
    "email": "user@example.com"
  }
}
```

> `name` は `options.data` で渡すことで、`profiles` 自動作成トリガーが `profiles.name` に書き込む

---

### 1-2. ログイン

```
POST /auth/v1/token?grant_type=password
```

**Request Body**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response `200`**
```json
{
  "access_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 3600,
  "refresh_token": "...",
  "user": {
    "id": "uuid",
    "email": "user@example.com"
  }
}
```

---

### 1-3. ログアウト

```
POST /auth/v1/logout
Authorization: Bearer {JWT_TOKEN}
```

**Response `204`** No Content

---

### 1-4. トークンリフレッシュ

```
POST /auth/v1/token?grant_type=refresh_token
```

**Request Body**
```json
{
  "refresh_token": "..."
}
```

**Response `200`** — 1-2 と同じ形式

---

## 2. Profiles API

### 2-1. 自分のプロフィール取得

```
GET /rest/v1/profiles?id=eq.{user_id}&select=*
Authorization: Bearer {JWT_TOKEN}
```

**Response `200`**
```json
[
  {
    "id": "uuid",
    "name": "ユーザー名",
    "avatar_url": "https://...",
    "is_premium": false,
    "created_at": "2026-01-01T00:00:00Z"
  }
]
```

---

## 3. Rooms API

### 3-1. ルーム一覧取得

```
GET /rest/v1/rooms?select=*&order=created_at.desc
```

> 未ログインでも閲覧可能（RLS: public read）

**Query Parameters（任意）**

| パラメータ | 例 | 説明 |
|---|---|---|
| `content_type` | `eq.image` | image / music でフィルタ |
| `room_type` | `eq.battle` | free / battle / quiz / collaboration |
| `limit` | `20` | 取得件数（デフォルト推奨: 20） |
| `offset` | `0` | ページネーション |

**Response `200`**
```json
[
  {
    "id": "uuid",
    "title": "夏の風景バトル",
    "description": "AIで夏の風景を生成して競おう",
    "base_prompt": "summer landscape, photorealistic",
    "room_type": "battle",
    "content_type": "image",
    "created_at": "2026-01-01T00:00:00Z"
  }
]
```

---

### 3-2. ルーム詳細取得

```
GET /rest/v1/rooms?id=eq.{room_id}&select=*
```

**Response `200`** — 3-1 の単一要素と同じ形式

---

## 4. Contents API

> `image_contents` と `music_contents` は同一の構造。
> 以下は `image_contents` で説明。`music_contents` も同様（`duration` フィールドが追加される）。

### status 定義

| status | 説明 | 可視範囲 |
|---|---|---|
| `generating` | Edge Function が生成処理中 | 自分のみ（RLS） |
| `pending` | 生成完了・ユーザーの投稿確定待ち | 自分のみ（RLS） |
| `completed` | 投稿済み・ルームに公開 | 全員 |
| `failed` | 生成失敗 / やり直し選択 | 自分のみ（RLS） |

---

### 4-1. ルームのコンテンツ一覧取得

```
GET /rest/v1/image_contents
  ?room_id=eq.{room_id}
  &select=*,likes(count),profiles(name,avatar_url)
  &order=created_at.desc
Authorization: Bearer {JWT_TOKEN}
```

> RLS により `completed` は全員が、`generating` / `pending` は本人のみ取得できる

**Response `200`**
```json
[
  {
    "id": "uuid",
    "user_id": "uuid",
    "room_id": "uuid",
    "file_url": "https://...supabase.co/storage/v1/object/public/images/...",
    "prompt_used": "海辺で夕日が沈む場面...",
    "status": "completed",
    "created_at": "2026-01-01T00:00:00Z",
    "likes": [{ "count": 12 }],
    "profiles": {
      "name": "ユーザー名",
      "avatar_url": "https://..."
    }
  }
]
```

---

### 4-2. コンテンツ詳細取得

```
GET /rest/v1/image_contents
  ?id=eq.{content_id}
  &select=*,likes(count),comments(*,profiles(name,avatar_url))
Authorization: Bearer {JWT_TOKEN}
```

---

### 4-3. 投稿確定 / やり直し（pending → completed / failed）

> 生成結果画面で「投稿する」「やり直す」をタップしたとき。
> `image_contents` / `music_contents` は RLS で UPDATE が許可されていないため、
> service_role を持つ Edge Function 経由で status を更新する。

```
POST /functions/v1/patch-content-status
Authorization: Bearer {JWT_TOKEN}
Content-Type: application/json
```

**Request Body**
```json
{
  "content_id": "uuid",
  "content_type": "image",
  "status": "completed"
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `content_id` | uuid | 対象コンテンツの ID |
| `content_type` | string | `image` / `music` |
| `status` | string | `completed`（投稿確定）/ `failed`（やり直し） |

**遷移ルール**
- 対象レコードの `user_id` が呼び出し元と一致しない場合は `403`
- 対象レコードが存在しない場合は `404`
- 対象レコードの現在 status が `pending` 以外の場合は `409`

**Response `200`**
```json
{ "success": true }
```

---

### 4-4. music_contents 固有フィールド

`music_contents` は上記に加えて `duration`（秒数・integer）が含まれる。

```json
{
  "duration": 30
}
```

---

## 5. Likes API

### 5-1. いいね追加

```
POST /rest/v1/likes
Authorization: Bearer {JWT_TOKEN}
Content-Type: application/json
Prefer: return=representation
```

**Request Body**
```json
{
  "user_id": "uuid",
  "content_type": "image",
  "content_id": "uuid"
}
```

**Response `201`**
```json
{
  "id": "uuid",
  "user_id": "uuid",
  "content_type": "image",
  "content_id": "uuid",
  "created_at": "2026-01-01T00:00:00Z"
}
```

> `(user_id, content_type, content_id)` に UNIQUE 制約あり。重複時は `409 Conflict`

---

### 5-2. いいね削除

```
DELETE /rest/v1/likes
  ?user_id=eq.{user_id}
  &content_type=eq.{content_type}
  &content_id=eq.{content_id}
Authorization: Bearer {JWT_TOKEN}
```

**Response `204`** No Content

---

### 5-3. 自分のいいね状態確認

```
GET /rest/v1/likes
  ?user_id=eq.{user_id}
  &content_type=eq.{content_type}
  &content_id=eq.{content_id}
  &select=id
Authorization: Bearer {JWT_TOKEN}
```

> 空配列 `[]` なら未いいね、レコードあれば いいね済み

---

## 6. Comments API

### 6-1. コメント一覧取得

```
GET /rest/v1/comments
  ?content_type=eq.{content_type}
  &content_id=eq.{content_id}
  &select=*,profiles(name,avatar_url)
  &order=created_at.asc
Authorization: Bearer {JWT_TOKEN}
```

**Response `200`**
```json
[
  {
    "id": "uuid",
    "user_id": "uuid",
    "content_type": "image",
    "content_id": "uuid",
    "body": "すごい！",
    "created_at": "2026-01-01T00:00:00Z",
    "profiles": {
      "name": "ユーザー名",
      "avatar_url": "https://..."
    }
  }
]
```

---

### 6-2. コメント追加

```
POST /rest/v1/comments
Authorization: Bearer {JWT_TOKEN}
Content-Type: application/json
Prefer: return=representation
```

**Request Body**
```json
{
  "user_id": "uuid",
  "content_type": "image",
  "content_id": "uuid",
  "body": "すごい！"
}
```

**Response `201`** — 作成後のレコード（`profiles` なし）

---

## 7. Edge Functions

### 7-1. generate-image

```
POST /functions/v1/generate-image
Authorization: Bearer {JWT_TOKEN}
Content-Type: application/json
```

**Request Body**
```json
{
  "room_id": "uuid",
  "prompt": "海辺で夕日が沈む場面、波が打ち寄せる砂浜..."
}
```

**処理フロー**

```
① ai_usage_logs を確認（当月の使用回数チェック）
  └─ 超過 → 429 usageLimitExceeded を返す

② image_contents レコードを status='generating' で INSERT

③ base_prompt + user_prompt を結合して Hugging Face API を呼び出す

④ 生成結果の画像を Storage に保存
   └─ images/{room_id}/{content_id}.png

⑤ image_contents を UPDATE
   └─ file_url = Storage URL
   └─ status = 'pending'
   └─ prompt_used = 結合後プロンプト

⑥ ai_usage_logs に INSERT（ログ記録）

⑦ content_id を返す
```

**Response `200`**
```json
{
  "content_id": "uuid"
}
```

**Error Responses**

| status | type | 説明 |
|---|---|---|
| `401` | `unauthorized` | 未ログイン |
| `429` | `usageLimitExceeded` | 月間生成回数上限超過 |
| `503` | `networkError` | Hugging Face API 接続失敗 |
| `500` | `serverError` | その他サーバーエラー |

---

### 7-2. generate-music

```
POST /functions/v1/generate-music
Authorization: Bearer {JWT_TOKEN}
Content-Type: application/json
```

**Request Body**
```json
{
  "room_id": "uuid",
  "prompt": "落ち着いたジャズ、ピアノとベース..."
}
```

**処理フロー**

```
① ai_usage_logs を確認（当月の使用回数チェック）
  └─ 超過 → 429 usageLimitExceeded を返す

② music_contents レコードを status='generating' で INSERT

③ base_prompt + user_prompt を結合して MusicGen API を呼び出す（最大 30 秒）

④ 生成結果の音声を Storage に保存
   └─ music/{room_id}/{content_id}.mp3

⑤ music_contents を UPDATE
   └─ file_url = Storage URL
   └─ duration = 生成秒数
   └─ status = 'pending'
   └─ prompt_used = 結合後プロンプト

⑥ ai_usage_logs に INSERT（ログ記録）

⑦ content_id を返す
```

**Response `200`**
```json
{
  "content_id": "uuid"
}
```

**Error Responses** — 7-1 と同様

---

### 7-3. check-usage-limit

```
GET /functions/v1/check-usage-limit
Authorization: Bearer {JWT_TOKEN}
```

**処理**

```
ai_usage_logs WHERE user_id = {jwt_user_id}
  AND used_at >= 月初 (date_trunc('month', now()))
  をカウント
```

**Response `200`**
```json
{
  "used": 3,
  "limit": 10,
  "is_premium": false,
  "remaining": 7
}
```

> `is_premium = true` の場合、`limit` は別途設定値（無制限 or 上限値）を返す

---

### 7-4. delete-account

> 設定画面の退会フローから呼び出す。アカウントと関連データを完全に削除する。

```
POST /functions/v1/delete-account
Authorization: Bearer {JWT_TOKEN}
Content-Type: application/json
```

**Request Body**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

> 本人確認のため、ログイン中ユーザーのメールアドレスとパスワードを再送する。

**処理フロー**

```
① JWT を検証し user を特定
  └─ 未ログイン / 失効 → 401 unauthorized

② リクエストの email が JWT の user.email と一致するか確認
  └─ 不一致 → 401 invalidCredentials

③ email + password で再認証（signInWithPassword）
  └─ 失敗 → 401 invalidCredentials

④ Storage の images バケットから {user_id}/ 配下のファイルを一覧取得し一括削除
   └─ images/{user_id}/{content_id}.jpg

⑤ auth.admin.deleteUser(user_id) を実行（service_role）
   └─ profiles を起点に image_contents / music_contents /
      likes / comments / ai_usage_logs が ON DELETE CASCADE で連鎖削除

⑥ success を返す（クライアントはローカルセッションを破棄）
```

**Response `200`**
```json
{
  "success": true
}
```

**Error Responses**

| status | type | 説明 |
|---|---|---|
| `401` | `unauthorized` | 未ログイン / JWT 期限切れ |
| `401` | `invalidCredentials` | メール不一致 / パスワード再認証失敗 |
| `500` | `serverError` | Storage 削除またはユーザー削除に失敗 |

---

## 8. Realtime 購読

> Edge Function からの DB 更新を受け取りクライアントの状態を遷移させる

### 8-1. 自分のコンテンツ生成完了を監視

```
channel: content-{content_id}
table:   image_contents  /  music_contents
event:   UPDATE
filter:  id=eq.{content_id}
```

**受信するペイロード（変更後レコード）**
```json
{
  "id": "uuid",
  "status": "pending",
  "file_url": "https://...supabase.co/storage/v1/object/public/images/..."
}
```

> `status` が `generating → pending` に変化したタイミングで生成結果画面へ遷移する

---

## 9. Storage URL 形式

| コンテンツ | パス |
|---|---|
| 画像 | `images/{room_id}/{content_id}.png` |
| 音楽 | `music/{room_id}/{content_id}.mp3` |

**公開 URL**
```
https://{PROJECT_REF}.supabase.co/storage/v1/object/public/{bucket}/{path}
```

> **実装上の注記**：現行のデプロイ済み実装では画像の保存パスはユーザー ID 起点の
> `images/{user_id}/{content_id}.jpg`（バケットは `images` のみ・public）。退会
> （7-4 delete-account）ではこの `{user_id}/` プレフィックス配下を削除対象とする。
> `music` バケットは未作成のため、音楽 Storage を追加した際は同様に `{user_id}/`
> 配下を退会時の削除対象に加える。
