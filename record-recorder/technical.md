# Record Recorder — 技術的な工夫

## アーキテクチャ方針

- **外部依存ゼロ**: CocoaPods / SPM を使わず、Apple 純正フレームワークのみで完結
- **運用コストゼロ**: Discogs API は認証なし・無料枠。バックエンドサーバー不要
- **iOS 17+ 専用**: `@Observable`, `@Bindable`, SwiftData など最新APIを積極採用

---

## 認識パイプライン

ジャケット撮影 or バーコードスキャンからアルバム情報を自動取得するパイプライン。

```
カメラ撮影
    └─ UIImage.fixedOrientation()
    └─ resized(maxDimension: 1200)
         ├─ [OCR path]
         │    └─ OCRService（VNRecognizeTextRequest）
         │    └─ DiscogsService.searchByText()
         └─ [Barcode path]
              └─ BarcodeService（VNDetectBarcodesRequest: EAN-13/8, UPC-E）
              └─ DiscogsService.searchByBarcode()
                   └─ 候補1件 → 自動入力
                   └─ 候補複数 → DiscogsResultView（ピッカー表示）
```

すべてオンデバイス処理のため、OCR・バーコード検出はオフライン動作可能。

### RecognitionService（actor）

OCR / バーコードのオーケストレーションを担う `actor`。  
`actor` にすることでデータ競合を防ぎつつ、非同期処理を安全に扱える。

---

## SwiftData + CloudKit の統合

### 制約と対応

CloudKit は `init` 引数なしのモデル初期化を要求するため、**全プロパティにデフォルト値が必要**。

```swift
// CloudKit 互換のための宣言パターン
@Model class Record {
    var title: String = ""
    var artist: String = ""
    var isFavorite: Bool = false
    var year: Int? = nil        // Optional は nil がデフォルト値
    // ...
}
```

### ModelContainer の設定

```swift
let config = ModelConfiguration(cloudKitDatabase: .automatic)
let container = try ModelContainer(for: Record.self, configurations: config)
```

`.automatic` により、iCloud アカウントがあれば自動同期・なければローカルのみ動作する。

---

## カメラモード設計

3つのモードを `CameraMode` enum で管理し、責務を明確に分離した。

```swift
enum CameraMode {
    case cover          // ジャケット保存用（ガイド表示・6:5クロップ）
    case coverNoGuide   // OCR用（ガイドなし・保存なし）
    case barcode        // バーコード検出用
}
```

`.coverNoGuide` で撮影した画像はジャケット写真として**保存しない**。  
OCR・Discogs 検索のみに使用し、結果だけをフォームに反映する。

---

## ボトムバーの実装

### `.safeAreaInset` を選んだ理由

`ToolbarItemGroup(.bottomBar)` は UIKit の Toolbar を内部で使用しており、SwiftUI との混在で警告が出た。  
代わりに `.safeAreaInset(edge: .bottom)` で純粋な SwiftUI 実装とした。

### iPhone 横向き / Duo 対応

`vSizeClass == .compact`（iPhone 横向き・Duo アウターディスプレイ）のとき、ボトムバーを trailing サイドバーへ自動切替する `FloatingBarModifier` を実装。

```swift
struct FloatingBarModifier: ViewModifier {
    @Environment(\.verticalSizeClass) var vSizeClass

    func body(content: Content) -> some View {
        if vSizeClass == .compact {
            content.safeAreaInset(edge: .trailing) {
                FloatingBar(isLandscape: true, ...)
            }
        } else {
            content.safeAreaInset(edge: .bottom) {
                FloatingBar(isLandscape: false, ...)
            }
        }
    }
}
```

`VerticalEdge` と `HorizontalEdge` は別型のため三項演算子では書けず、`if/else` による分岐が必要。

---

## 検索バーのカスタム実装

`List.searchable()` は検索バーの位置・デザインを制御できないため廃止し、`ScrollView` のコンテンツ内にカスタム `TextField` を配置した。

**メリット**:
- スクロールダウンで自然に隠れる（ステータスバータップでトップへ戻る標準挙動も機能する）
- 角丸をカード（16pt）と統一できる
- `.searchable()` 特有の NavigationStack 依存がなくなる

---

## Modal 管理

`AddRecordView` では複数の sheet が競合しないよう、`AddRecordSheet` enum を使った**単一の `.sheet(item:)`** で管理している。

```swift
enum AddRecordSheet: Identifiable {
    case discogsResult([DiscogsRelease])
    // 追加の sheet ケースをここに集約
}
```

カメラは `.fullScreenCover(item:)` を使用（fullScreenCover の中で fullScreenCover を重ねられるため）。
