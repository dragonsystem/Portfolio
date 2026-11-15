# SUTRA — 技術的な工夫

## 構成

```
code.ts           メインの処理。Figma Plugin API（figma.*）はここだけで使う
ui.html / ui.ts   プラグインの画面
tokenStoreUtils.ts Figma API に依存しない計算関数（Vitest でテスト）
```

Figma のプラグインは、キャンバスを操作する本体（code.ts）と画面（ui.html）が別の環境で動き、`postMessage` でやりとりする。
ui.html はバンドルの対象外で `import` が使えないため、計算は code.ts 側で行い、結果だけを画面へ送る形にそろえている。

---

## 1 色からトーンを自動計算する

Token Store でユーザーが選ぶのは、各カテゴリのアンカー色 1 色だけ。
そこから Material Design 3 に近い考え方で、10〜23 段階の明るさの色（トーン）を自動で計算する。

```
アンカー色（H / S / L）
    └─ toneToRgb(palette, tone)
         ├─ tone 50 = アンカー色
         ├─ Default / Hover / Pressed
         ├─ Container（薄い背景色）
         └─ その上に載せる文字色
```

| 色の種類 | 明るさの決め方 |
|---|---|
| アクションカラー（primary / secondary / brand-main / brand-sub） | ユーザーが選んだアンカーの明るさを基準にする |
| ステータスカラー（error / warning / success / info） | 役割ごとに決めたトーンの段階で固定 |

この計算ロジックはコードの中にしか存在しない。プラグインが使えなくなっても再現できるよう、計算式を仕様書として別に残した。

---

## Figma Variables との同期

トークンを Figma の Variables として書き出し、生成するコンポーネントの色・間隔・角丸に紐づける。

```
Sutra Primitives               … 色の実際の値（color/{palette}/{tone}）
Sutra Semantic Color Themes    … 役割ごとの色（primary/default など）。Light / Dark の 2 モード
Sutra Spacing / Radius / ...   … 間隔・角丸・文字サイズ
```

Semantic の変数は Primitive の変数を参照する（エイリアス）。テーマを Dark に切り替えると、配置済みのコンポーネントも追従する。

### 紐づけが外れる不具合

コンポーネントを生成したとき、一部の色が Variables に紐づかず、固定の値で描かれることがあった（Dark モードに追従しない）。
原因は、Variables の対応表（`VAR_MAP`）を読み込み終わる前に描画が始まっていたこと。
各生成処理の最初で必ず `await ensureVariables()` を実行し、対応表ができてから描画するようにした。

---

## 手動変更を守る：競合の検知

トークンを保存すると Variables が上書きされる。しかし利用者が Figma 上で Variables を手で調整していることもある。
そこで、**Sutra が前回書き込んだ値**を各 Variable に記録しておき、保存時に比較する。

```typescript
// Variable の pluginData に、最後に書き込んだ値を保存
variable.setPluginData("sutra_last_written_value", JSON.stringify(snapshot));
```

```
保存ボタン
  → 現在の値と「前回 Sutra が書いた値」を比べる
  → 違う = 誰かが手で変えた（競合）
  → 競合があるときだけ一覧を表示し、項目ごとに「上書き / 残す」を選ぶ
```

- 内部の層（Primitive / Semantic）の区別は画面に出さず、すべて同じ形式の一覧で見せる
- 当初はリセットボタンだけ確認なしで上書きしていた。「保存では守るのにリセットでは黙って消える」のは一貫しないと指摘を受け、リセットも同じ確認の流れにそろえた

---

## コントラストの自動検証

ユーザーが色を変えると、文字と背景のコントラストが足りなくなることがある。保存時に、決めておいた文字色と背景色の組み合わせをすべて計算して確かめる。

```
相対輝度:     L = 0.2126R + 0.7152G + 0.0722B（R, G, B はガンマ補正後の値）
コントラスト比: (L1 + 0.05) / (L2 + 0.05)
```

WCAG AA を下回る組み合わせがあれば警告する。それでも適用する場合は、理由を記録できる。

---

## Figma API の制約への対処

Figma の仕様が原因の制約は、回避策を勝手に作らず、一覧にして管理している。

| 制約 | 内容 |
|---|---|
| 影の不透明度 | Effect の `opacity` を Variable に紐づける項目がない |
| REST API | Variables の REST API は Enterprise プランのみ |
| モード数 | Variables のモード数はプランによって上限が違う |
| フォント | フォントを読み込む前に値を設定すると例外が出る |

あわせて、Figma の「ページを必要なときだけ読み込む」方式（Dynamic Page Loading）に合わせ、生成処理を非同期に作り直した。

---

## 書き出し

| ファイル | 内容 |
|---|---|
| `sutra-tokens.json` | W3C Design Tokens 形式のトークン |
| `styles.css` | CSS カスタムプロパティ |
| `guidelines.md` | デザイナー・エンジニア向けのスタイルガイド |
| `sutra-component-reference.md` | コンポーネントの仕様書 |
| Code Connect スタブ | Figma のコンポーネントと実装コードを結びつけるための雛形 |

W3C 形式の JSON は読み込みにも対応しており、書き出したトークンを別のファイルで再現できる。
