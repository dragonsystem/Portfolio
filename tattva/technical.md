# TATTVA — 技術的な工夫

## 生成パイプライン：API 呼び出しを 2 回に分ける

当初は 1 回の呼び出しで両方を出力させ、`===DESIGN.MD===` / `===BLUEPRINT.MD===` という区切り文字で分割していた。
しかし BLUEPRINT.md がまったく生成されないことがあった。

**原因**

- DESIGN.md だけで出力の上限（65,536 トークン）に届き、BLUEPRINT に入る前に打ち切られる
- 区切り文字を守るかどうかが AI の出力次第で、毎回同じ結果にならない

| 案 | 内容 | 評価 |
|---|---|---|
| A | API 呼び出しを 2 回に分ける | 確実に分離できる |
| B | Gemini の構造化出力（JSON） | JSON と Markdown の相性に不安 |
| C | プロンプトの強化のみ | 根本的な解決にならない |

API のコストより確実さを優先して **案 A** を採用した。

```
1回目: DESIGN.md 用プロンプト + 画像 → designOutput
         ↓ 1回目の全文を {{design_output}} として渡す
2回目: BLUEPRINT.md 用プロンプト + 画像 → blueprintOutput
```

2 回目に 1 回目の結果を渡すことで、2 つのファイルの数値がずれないようにしている。

---

## API エラーへの対応

### 自動リトライ

`callWithRetry()` で 503・429・unavailable を検出したとき、2 秒 → 4 秒の指数バックオフで最大 3 回再試行する。再試行中は「自動リトライ中... (1/3)」と表示する。

### 失敗したボタンを Retry に変える

すべて失敗したときは、専用の再試行ボタンを増やすのではなく、**失敗したパターンのボタンだけ**ラベルを「Retry Precision」などに変える。

```
A Precision → 押下・失敗 → Retry Precision
Retry Precision → 押下・成功 → A Precision
```

- ボタンが増えないので UI が散らからない
- どのパターンが失敗したかがボタンの位置ですぐわかる
- 成功すれば元のラベルに戻る

---

## App Sandbox でのファイル保存

### 問題：保存したはずのファイルがない

保存先フォルダを設定してもファイルが保存されていなかった。

- サンドボックスでは、`NSOpenPanel` でユーザーが選んだ URL のアクセス権の範囲でしか書き込めない
- パスを文字列で保存して `URL(fileURLWithPath:)` で作り直していたため、アクセス権が失われていた
- `try?` でエラーを握りつぶしていたので、失敗しても何も表示されなかった

### 対処：セキュリティスコープブックマーク

```swift
// 選んだ URL からブックマークを作って保存（再起動後も使える）
settings.setSavePath(from: panel.url)

// 書き込み時はアクセス権を明示的に開始・終了する
guard baseURL.startAccessingSecurityScopedResource() else { ... }
defer { baseURL.stopAccessingSecurityScopedResource() }
```

### 審査での指摘（Guideline 2.4.5(i)）

初回起動時に ~/Downloads のブックマークを自動で登録していたが、これでは実際のアクセス権は得られず、保存は失敗していた。
最初に保存が必要になった時点で `NSOpenPanel` を出してユーザーにフォルダを選んでもらう方式に変え、書き込みに失敗したときはアラートで知らせるようにした。

---

## mermaid ダイアグラムの描画

### サンドボックスで CDN が読めない

mermaid.js を CDN から読み込んでいたため、サンドボックス環境では描画されなかった。
mermaid.min.js（約 3.3MB）をアプリに同梱し、`Bundle.main` から読み込むよう変えた。

### 構文エラーへの対処

日本語と括弧を含むラベル（例：`A[カテゴリ (興味あるもの)]`）で、mermaid の構文エラーが起きていた。対処は 3 段階。

1. **サニタイザー**：描画前に `[ラベル(括弧)]` を `["ラベル(括弧)"]` に自動変換する `sanitizeMermaid()`
2. **プロンプトの制約**：ラベルは `["..."]` で囲む、ノード ID は英数字とアンダースコアのみ、などのルールを AI に指示
3. **エラー通知**：WKWebView の JavaScript から Swift へメッセージを送り、描画に失敗した図に ⚠ を表示

### ズーム・パン・書き出し

- スクロールでカーソル位置を中心にズーム、ドラッグでパン（WKWebView 内の JavaScript で実装）
- ズーム範囲は 2%〜2000%。ボタン 1 回の変化量を「現在の倍率の 30%」にして、どの倍率でも操作感が同じになるようにした
- `WKWebView.takeSnapshot` で表示中の図を PNG として書き出す

---

## Google Stitch 適合度チェック

`DesignValidator.swift` で、生成した DESIGN.md を TATTVA 独自のルールでチェックする。

- トップレベルの 10 キーがあるか
- カラートークン 20 キーがあり、hex / rgba の形式が正しいか
- タイポグラフィ 4 ロール × 5 項目がそろっているか、禁止された文字列がないか
- shadows・transitions・rounded・components の必須キー
- コンポーネントに hex 値が直接書かれていないか（トークンを参照すべき箇所）

結果は「エラー」ではなく**「改善推奨」「確認推奨」**と表示する。生成結果は使えないものではなく、品質に差があるだけ、という考え方による。

---

## マルチウィンドウ

`SettingsManager`・`ContextTemplateManager`・`GenerationHistoryManager` をアプリ全体で 1 つにし、`.environmentObject()` で全ウィンドウに渡している。

| ウィンドウごと | 全ウィンドウ共通 |
|---|---|
| 画像・Context・生成結果 | 設定・テンプレート・履歴 |

---

## API キーの保存

API キーは UserDefaults（平文）ではなく macOS Keychain に保存している。
