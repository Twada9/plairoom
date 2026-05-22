# 技術設計書

## バックエンド

### サービス：Supabase

| 機能 | 用途 |
|---|---|
| Database（PostgreSQL） | データ永続化 |
| Auth | ユーザー認証 |
| Storage | 画像・音楽ファイル保存 |
| Edge Functions | AI生成API呼び出し・回数制限チェック |
| Realtime | 生成完了通知（検討中） |

### Storage 構成

```
storage/
├── images/
│   └── {room_id}/{content_id}.png
└── music/
    └── {room_id}/{content_id}.mp3
```

### Edge Functions 一覧

| 関数名 | 役割 |
|---|---|
| generate-image | 画像生成AIを呼び出し、結果をStorageに保存 |
| generate-music | 音楽生成AIを呼び出し、結果をStorageに保存 |
| check-usage-limit | 無料ユーザーの生成回数を確認 |
| delete-account | 退会処理。パスワード再認証後、Storageファイルを削除し auth.users を削除（DBは CASCADE で連鎖削除） |

### AI サービス

**方針：無料枠あり、または無料で使えるAPIを使用する**

| コンテンツ | 候補 | 状態 |
|---|---|---|
| 画像生成 | Hugging Face Inference API（FLUX.1 / Stable Diffusion XL）/ Replicate | 候補選定中 |
| 音楽生成 | MusicGen（Hugging Face Inference API）/ Suno API | 候補選定中 |

> ※ 無料枠の範囲・レート制限・出力品質を比較して最終選定する

---

## Android

### アーキテクチャ：MVVM

```
UI（Composable）
  ↓
ViewModel
  ↓
Repository
  ↓
APIクライアント（Retrofit + OkHttp）/ Room
```

### 使用技術

| 技術 | 用途 |
|---|---|
| Jetpack Compose | UI |
| Koin | DI（依存性注入） |
| OkHttp | HTTPクライアント |
| Retrofit | Supabase REST API との通信 |
| Room | ローカルキャッシュ（必要になれば） |

### 層の責務

| 層 | 責務 |
|---|---|
| UI（Composable） | 画面表示・ユーザー操作の受付 |
| ViewModel | UIStateの管理・Repositoryの呼び出し |
| Repository | データソースの抽象化・キャッシュ戦略 |
| APIクライアント | Supabase REST API との通信処理 |
| Entity | データモデル定義 |

### ディレクトリ構成

```
app/
├── ui/
│   ├── roomlist/
│   │   ├── RoomListScreen.kt
│   │   └── RoomListViewModel.kt
│   ├── roomdetail/
│   │   ├── RoomDetailScreen.kt
│   │   └── RoomDetailViewModel.kt
│   ├── generate/
│   │   ├── GenerateScreen.kt
│   │   └── GenerateViewModel.kt
│   ├── result/
│   │   ├── ResultScreen.kt
│   │   └── ResultViewModel.kt
│   ├── settings/
│   │   ├── SettingsScreen.kt
│   │   └── SettingsViewModel.kt
│   └── auth/
│       ├── AuthScreen.kt
│       └── AuthViewModel.kt
├── repository/
│   ├── RoomRepository.kt
│   ├── ContentRepository.kt
│   ├── AuthRepository.kt
│   ├── LikeRepository.kt
│   └── CommentRepository.kt
├── api/
│   ├── SupabaseApiClient.kt
│   └── dto/
│       ├── RoomDto.kt
│       ├── ImageContentDto.kt
│       └── MusicContentDto.kt
├── entity/
│   ├── Room.kt
│   ├── ImageContent.kt
│   ├── MusicContent.kt
│   ├── Like.kt
│   └── Comment.kt
└── di/
    └── AppModule.kt
```

### 退会フロー（Settings）

```
SettingsScreen（退会ボタン）
  → 退会確認 UI（削除内容の警告 + パスワード入力ダイアログ）
  → SettingsViewModel.deleteAccount(password)
  → AuthRepository.deleteAccount(email, password)
      → delete-account Edge Function を呼び出し
  → 成功時：ローカルセッションを破棄しログイン前状態へ遷移
```

---

## iOS

### アーキテクチャ：TCA（The Composable Architecture）✅ 確定

### 使用技術

| 技術 | 用途 |
|---|---|
| SwiftUI | UI |
| TCA（swift-composable-architecture） | アーキテクチャ・状態管理 |
| swift-dependencies | DI（依存性注入） |
| Combine | 非同期処理（必要になれば） |

### 層の責務

| 層 | 責務 |
|---|---|
| View（SwiftUI） | 画面表示・ユーザー操作の受付 |
| Reducer（Feature） | State・Action の定義・ビジネスロジック |
| Store | State の保持・Action のディスパッチ |
| Repository | データソースの抽象化 |
| APIクライアント | Supabase Swift SDK のラッパー |
| Entity | データモデル定義 |

### ディレクトリ構成

```
App/
├── Features/
│   ├── RoomList/
│   │   ├── RoomListView.swift
│   │   └── RoomListFeature.swift      // Reducer・State・Action
│   ├── RoomDetail/
│   │   ├── RoomDetailView.swift
│   │   └── RoomDetailFeature.swift
│   ├── Generate/
│   │   ├── GenerateView.swift
│   │   └── GenerateFeature.swift
│   ├── Result/
│   │   ├── ResultView.swift
│   │   └── ResultFeature.swift
│   ├── Settings/
│   │   ├── SettingsView.swift
│   │   └── SettingsFeature.swift
│   └── Auth/
│       ├── AuthView.swift
│       └── AuthFeature.swift
├── Repository/
│   ├── RoomRepository.swift
│   ├── ContentRepository.swift
│   ├── AuthRepository.swift
│   ├── LikeRepository.swift
│   └── CommentRepository.swift
├── API/
│   ├── SupabaseAPIClient.swift
│   └── DTO/
│       ├── RoomDTO.swift
│       ├── ImageContentDTO.swift
│       └── MusicContentDTO.swift
├── Entity/
│   ├── Room.swift
│   ├── ImageContent.swift
│   ├── MusicContent.swift
│   ├── Like.swift
│   └── Comment.swift
└── DI/
    └── Dependencies.swift
```

### 退会フロー（Settings）

```
SettingsView（退会ボタン）
  → 退会確認 UI（削除内容の警告 + パスワード入力アラート）
  → SettingsFeature: Action.deleteAccountTapped / .reauthSubmitted(password)
  → AuthRepository.deleteAccount(email, password)
      → delete-account Edge Function を呼び出し
  → 成功時：ローカルセッションを破棄しログイン前状態へ遷移
```

---

## 未確定事項

- 画像生成AIサービスの最終選定（候補：Hugging Face / Replicate）
- 音楽生成AIサービスの最終選定（候補：MusicGen / Suno API）
- 生成完了通知の実装方法（Realtime / Push通知）
- 無料ユーザーの生成回数上限
- Room のローカルキャッシュ戦略（Android）
