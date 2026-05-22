# TCA (The Composable Architecture) セットアップ手順

## 1. Swift Packageの追加

### Xcodeでパッケージを追加する

1. **Xcodeでプロジェクトを開く**
   ```bash
   open PLAIROOM-iOS.xcodeproj
   ```

2. **Package Dependenciesを開く**
   - プロジェクトナビゲータで `PLAIROOM-iOS` プロジェクトを選択
   - **Package Dependencies** タブを開く

3. **TCAパッケージを追加**
   - 「+」ボタンをクリック
   - 検索バーに以下のURLを入力:
     ```
     https://github.com/pointfreeco/swift-composable-architecture
     ```
   - **Dependency Rule**: `Up to Next Major Version` を選択
   - **Version**: `1.0.0` を指定（または最新バージョン）
   - **Add Package** をクリック

4. **ターゲットにライブラリを追加**
   - `ComposableArchitecture` を選択
   - **Add to Target**: `PLAIROOM-iOS` を選択
   - **Add Package** をクリック

5. **Dependencies ライブラリも追加**
   - 同じ手順で以下のパッケージも追加:
     ```
     https://github.com/pointfreeco/swift-dependencies
     ```
   - **Version**: `1.0.0`以降

## 2. 実装済みファイル

### DI層
- ✅ `PLAIROOM-iOS/DI/Dependencies.swift`
  - `SupabaseConfig` の DependencyKey
  - `SupabaseClient` の DependencyKey
  - `liveValue` / `testValue` / `previewValue` の定義

### API層
- ✅ `PLAIROOM-iOS/API/SupabaseConfig.swift`
  - `static let shared` を削除（DI化完了）

- ✅ `PLAIROOM-iOS/API/SupabaseClient.swift`
  - `init(config:)` に変更（デフォルト引数削除）

## 3. 使用方法

### TCA Featureでの使用例

```swift
import ComposableArchitecture

@Reducer
struct RoomListFeature {
    @Dependency(\.supabaseClient) var supabaseClient

    struct State: Equatable {
        var rooms: [Room] = []
        var isLoading = false
    }

    enum Action {
        case onAppear
        case roomsResponse(Result<[Room], Error>)
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .onAppear:
                state.isLoading = true
                return .run { send in
                    do {
                        let rooms: [Room] = try await supabaseClient.get(
                            endpoint: "/rooms",
                            queryItems: [URLQueryItem(name: "select", value: "*")]
                        )
                        await send(.roomsResponse(.success(rooms)))
                    } catch {
                        await send(.roomsResponse(.failure(error)))
                    }
                }

            case .roomsResponse(.success(let rooms)):
                state.isLoading = false
                state.rooms = rooms
                return .none

            case .roomsResponse(.failure):
                state.isLoading = false
                return .none
            }
        }
    }
}
```

### テストでの使用例

```swift
import XCTest
import ComposableArchitecture
@testable import PLAIROOM_iOS

final class RoomListFeatureTests: XCTestCase {
    func testFetchRooms() async {
        let store = TestStore(initialState: RoomListFeature.State()) {
            RoomListFeature()
        } withDependencies: {
            // テスト用のモック設定を注入
            $0.supabaseClient = SupabaseClient(
                config: SupabaseConfig(
                    projectRef: "test-project",
                    anonKey: "test-key"
                )
            )
        }

        await store.send(.onAppear) {
            $0.isLoading = true
        }

        // ...
    }
}
```

## 4. ビルド確認

```bash
xcodebuild -project PLAIROOM-iOS.xcodeproj \
  -scheme PLAIROOM-iOS \
  -configuration Debug \
  build \
  -destination 'platform=iOS Simulator,name=iPhone 17'
```

## 5. トラブルシューティング

### パッケージが見つからない

```bash
# Xcode派生データをクリア
rm -rf ~/Library/Developer/Xcode/DerivedData

# Xcodeを再起動
```

### ビルドエラー: Module 'Dependencies' not found

- Package Dependenciesタブで `swift-dependencies` が追加されているか確認
- パッケージを削除して再追加

### Actor分離警告

- `@MainActor` が正しく付与されているか確認
- `DependencyValues` の `liveValue` に `@MainActor` があるか確認

## 参考リンク

- [TCA公式ドキュメント](https://pointfreeco.github.io/swift-composable-architecture/)
- [Swift Dependencies](https://github.com/pointfreeco/swift-dependencies)
