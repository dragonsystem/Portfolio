# Marga — 技術的な工夫

## アーキテクチャ方針

- **外部依存ゼロ**: CocoaPods / SPM を使わず Apple 純正フレームワークのみで完結
- **運用コストゼロ**: CloudKit はユーザーの iCloud アカウントに紐付くため開発者負担なし
- **iOS 17+ 専用**: `@Observable`, SwiftData など最新 API を積極採用

---

## MapKit UIViewRepresentable パターン

SwiftUI の `Map` ではなく `MKMapView` を `UIViewRepresentable` でラップしている。  
SwiftUI の `Map` ではルートラインの描画やズーム制御など、細かいカスタマイズが困難なため。

### setRegion のタイミング問題

`.safeAreaInset` によるコンテンツ切替（RecordButton → バナー）が `MKMapView` のフレームを変更し、`setRegion` のアニメーションを上書きする問題が発生した。

```swift
// フレーム変更が落ち着くのを待ってから setRegion を呼ぶ
DispatchQueue.main.asyncAfter(deadline: .now() + 0.4) {
    mapView.setRegion(region, animated: true)
}
```

### フィードバックループの防止

`regionDidChangeAnimated` デリゲートを使うとビュー更新 → リージョン変更 → ビュー更新のループが発生するため削除。  
Coordinator で `lastSelectedMapItem` / `lastRecenterToSpotTrigger` を追跡し、不要な再実行を防止した。

### 再移動トリガーのトグルパターン

同じスポットへの再移動（バナーの再タップ）では値が変わらず `updateUIView` が反応しない問題を、Bool のトグルパターンで解決した。

```swift
// 同じスポットに再センタリングする場合もトグルで検出
recenterToSpotTrigger.toggle()
```

---

## レシート OCR パイプライン（給油記録）

Vision Framework の `VNRecognizeTextRequest` で日本語 + 英語を認識し、給油量・単価・店舗情報を自動抽出する。

```
カメラ撮影
    └─ VNRecognizeTextRequest（日本語 + 英語）
         ├─ 給油量・単価・合計金額の抽出
         ├─ 店舗名: 3パス方式
         │    Pass 1: ブランドキーワード一致（ENEOS, Shell 等）
         │    Pass 2: 店舗指標を含む行（「SS」「ガソリンスタンド」等）
         │    Pass 3: レシート先頭5行のフォールバック
         ├─ 支払い方法: 優先度5段階
         │    1. 現金（「お釣り」等の間接表現優先）
         │    2. 具体的決済ブランド（PayPay, Suica 等）
         │    3. クレジット一般
         │    4. マスクカード番号検出
         │    5. カード単体
         └─ 店舗住所: 都道府県 → 郵便番号 → 市区町村+番地
```

---

## SwiftData + CloudKit の統合

### CloudKit の制約と対処

| 制約 | 対処 |
|---|---|
| 全属性にデフォルト値が必要 | 全モデルのプロパティ宣言に `= defaultValue` を追加 |
| リレーションシップは Optional 必須 | `Ride.waypoints` / `spots` を `[T]?` に変更。アクセス時は `?? []` |
| `deleteRule: .cascade` 非対応 | `.nullify` に変更し、削除時に子オブジェクトを先に明示削除 |

### VersionedSchema の廃止

V1/V2/V3 が同一モデルクラスを参照すると "Duplicate version checksums detected." が発生した。  
今回の変更（デフォルト値追加・Optional 化）はすべて自動軽量マイグレーションの範囲内のため、`migrationPlan` を削除して `cloudKitDatabase: .automatic` のみで初期化。

```swift
let config = ModelConfiguration(cloudKitDatabase: .automatic)
let container = try ModelContainer(for: Ride.self, FuelLog.self, ..., configurations: config)
```

---

## JSON バックアップの UUID マージ設計

インポート時の挙動を REPLACE（全置換）ではなく **UUID マージ**にした。

- 端末移行時は新端末が空なので全件追加となり、実質的に REPLACE と同じ
- 誤って同じファイルを2回インポートしてもデータが重複しない

---

## Watch 連携の通知設計

iPhone 停止・Watch 停止のどちらも `RideRecorder.stop()` を通るため、`NotificationCenter.rideDidStop` の1点で両対応している。

```swift
// iPhone からも Watch からも同じメソッドを呼ぶ
func stop() {
    // ... データ保存 ...
    NotificationCenter.default.post(name: .rideDidStop, object: nil)
}
```

RideListView はこの通知を受けてフィルターをリセットし、最新の記録を必ず表示する。

---

## Modal 管理（長押しシート）

`.sheet(isPresented:)` + `if let coord = longPressCoord` の組み合わせでは、sheet 表示のタイミングで `coord` が nil になり空白シートが出るバグが発生した。  
`LongPressCoordinate: Identifiable` ラッパー構造体を作り、`.sheet(item:)` パターンに変更して解決。

```swift
struct LongPressCoordinate: Identifiable {
    let id = UUID()
    let coordinate: CLLocationCoordinate2D
}

// item が nil でない限り sheet が確実に値を持つ
.sheet(item: $longPressItem) { item in
    AddSpotView(coordinate: item.coordinate)
}
```
