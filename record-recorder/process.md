# Record Recorder — 開発プロセス

## タイムライン

```
2026-05-02〜05-10  Phase 0  機能修正・設計の確立
2026-05-12         Phase 1  Initial Commit（MVP完成）
2026-05-13         🚀 App Store 初回リリース
2026-06-01         Phase 2〜7  iCloud同期・統計・CSV・UI改善
2026-11-15         Phase 8〜9  .gitignore整備・iPhone横向き/Duo対応
```

---

## Phase 0 — 設計の確立（2026-05-02〜05-10）

### カメラモードの責務分離

ジャケット撮影（`.cover`）と自動入力用撮影（`.coverNoGuide`）を明確に分けた。

**きっかけ**: 自動入力用の撮影画像が意図せずジャケット写真として上書き保存されていたバグ。  
**解決**: `.coverNoGuide` からジャケット保存処理（1行）を削除し、OCR専用であることを明示。

| モード | 目的 | ジャケット保存 | OCR |
|---|---|---|---|
| `.cover` | ジャケット撮影 | ✅ | ❌ |
| `.coverNoGuide` | 自動入力用 | ❌ | ✅ |
| `.barcode` | バーコードスキャン | ❌ | ❌（バーコード検出） |

### SwiftData マイグレーション問題と解決

`isFavorite: Bool`（デフォルト値なし）を追加したところ、保存が一切できなくなった。

**原因**: SwiftData + CloudKit は全プロパティにデフォルト値が必要。  
**解決**: `var isFavorite: Bool = false` と宣言時にデフォルト値を付与。

```swift
// NG（保存不能）
var isFavorite: Bool

// OK
var isFavorite: Bool = false
```

---

## Phase 1 — MVP リリース（2026-05-12〜05-13）

### Initial Commit 時点での実装範囲

git 管理を開始した時点で以下がすべて実装済みだった：

- SwiftUI + SwiftData によるコレクション管理（LazyVGrid）
- Vision OCR / バーコード → Discogs API 自動認識パイプライン
- 3種カメラモード
- お気に入り・フィルター・ソート・検索
- ゴールドを基調としたデザインシステム
- アプリアイコン（通常・ダーク・Tinted の3バリアント）

### 写真クロップの統一

PhotoPicker からの横長画像がDetailViewのレイアウトを崩す問題が発覚。  
カメラ撮影はすでに 6:5 クロップ済みだったため、PhotoPicker 側にも `centerCropped(toRatio: 6.0/5.0)` を適用して統一した。

**2026-05-13 App Store 初回リリース**

---

## Phase 2〜7 — 機能拡充（2026-06-01）

### iCloud 同期（Phase 2）

`ModelConfiguration(cloudKitDatabase: .automatic)` の1行で自動同期を実現。  
ただし CloudKit 互換のため **全プロパティにデフォルト値が必要** という制約があり、モデル全体を修正した。

```swift
// Before
ModelConfiguration()

// After
ModelConfiguration(cloudKitDatabase: .automatic)
```

### 統計画面・CSV（Phase 3、5）

SwiftCharts でジャンル・年代の分布をグラフ表示。  
エクスポートは `ShareLink`、インポートは `fileImporter` で実装。  
重複検出はカタログ番号優先・なければタイトル＋アーティストの2段階で判定。

### フローティングピル型ボトムバー（Phase 4、6）

ユーザーフィードバックを受けてフルワイドバーからフローティングカプセルへ変更。  
`ToolbarItemGroup(.bottomBar)` は UIKit Toolbar 警告が出るため、`.safeAreaInset(edge: .bottom)` で実装した。

---

## Phase 8〜9 — メンテナンス・デバイス対応（2026-11-14〜11-15）

### iPhone Duo 横向き対応（Phase 9）

iPhone Duo のアウターディスプレイ（コンパクト縦サイズクラス）では `+` ボタンの位置が通常と異なるため、空コレクション時の案内文言をデバイス非依存に修正。

さらに `vSizeClass == .compact` を検出して、ボトムバーを trailing サイドバーへ自動切替する `FloatingBarModifier` を実装した。

---

## 意思決定ログ（主要なもの）

| 時期 | 問い | 選択 | 理由 |
|---|---|---|---|
| 05-02 | ジャケット撮影でOCRも走らせるか | 分離する | 責務を明確にするため |
| 05-04 | お気に入りのアイコン | 星（★）/ ゴールド | ハートより音楽ライブラリらしい |
| 05-31 | 検索バーの実装 | カスタム（ScrollView内） | `.searchable()` は位置・デザイン制御不可 |
| 05-31 | ボトムバーの形状 | フローティングカプセル | 画面領域を圧迫しない |
| 06-01 | iCloud UI の変更 | しない | SwiftData + CloudKit は自動同期のためUI不要 |
| 11-15 | 横向きボトムバー | trailing サイドバー | 親指が届きやすい位置に |
