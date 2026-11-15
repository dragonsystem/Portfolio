# Ryu Kaneko — Portfolio

UI/UX デザイナーが、AI の技術的進歩を体感・理解するために行った iOS / macOS アプリ個人開発のポートフォリオです。  
→ [プロフィール詳細](./profile.md)　｜　[連絡先](./profile.md#連絡先)

---

## アプリ

| プロジェクト | 概要 | 技術スタック | ステータス |
|---|---|---|---|
| [Record Recorder](./record-recorder/) | レコードコレクター向けコレクション管理アプリ | SwiftUI / SwiftData / Vision / CloudKit | [App Store リリース済み](https://apps.apple.com/us/app/record-recorder/id6766056660) |
| [Marga](./marga/) | バイクツーリング向け記録・管理アプリ | SwiftUI / SwiftData / MapKit / Vision / CloudKit / WatchConnectivity | [App Store リリース済み](https://apps.apple.com/us/app/marga/id6763526779) |
| [Suchi](./suchi/) | バイク・自転車向けコンパス型直線ナビ | SwiftUI / MapKit / CoreLocation / SwiftData / GeoAstro（自作） | [App Store リリース済み](https://apps.apple.com/us/app/suchi-compass-beeline-nav/id6792340914) |
| [TATTVA](./tattva/)（macOS） | 画像とテキストから AI でデザイン仕様書・UX 設計書を生成 | SwiftUI / Gemini API / WKWebView / mermaid.js / Keychain | [Mac App Store リリース済み](https://apps.apple.com/us/app/tattva/id6766077074?mt=12) |

---

## Figma プラグイン

| プロジェクト | 概要 | 技術スタック | ステータス |
|---|---|---|---|
| [SUTRA](./sutra/) | デザイントークンと連動する UI コンポーネント・ワイヤーフレーム生成 | TypeScript / Figma Plugin API / Variables / esbuild / Vitest | [Figma Community 公開中](https://www.figma.com/community/plugin/1652926105517784439/sutra)（React 出力を開発中） |
| [SATORI](./satori/) | 選択したフレームの構造を AI で解析し、ワンクリックで直す | TypeScript / Figma Plugin API / Gemini API | 組織内リリース |
| [SAKSHI](./sakshi/) | デザインがデザインシステムに沿っているかを監査する | TypeScript / Figma Plugin API / Variables / Gemini API | 組織内リリース |
| [OOUI プラグイン群](./ooui/)（Modeler / Modeler Pro / View View） | OOUI のモデリングからワイヤーフレーム生成までを Figma 上でつなぐ 3 本 | TypeScript / Figma Plugin API / FigJam / Gemini API / Vite | 組織内リリース |

---

## 設計の特徴：ツールの中から AI を呼ぶ

AI と別の画面で対話するのではなく、**作業しているツールの中から直接 AI を呼び出せる**ようにしている。
対象は TATTVA・SATORI・SAKSHI・Modeler・Modeler Pro の 5 本（Gemini API を使用）。

- **サーバーを持たない**：アプリやプラグインから Gemini API を直接呼ぶ。開発者がサーバーを運用する必要がなく、配布しやすい
- **API キーは利用者のもの**：各自のキーを設定画面で登録し、端末の中だけに保存する（macOS は Keychain、Figma は clientStorage）
- **AI に作業中のデータを渡す**：選んだフレームの画像とレイヤー構造、キャンバス上のモデルなどを渡すので、説明を書き写さなくても文脈をふまえた結果が返る
- **結果はツールの中で使える**：AI の出力は決まった形式（JSON）で受け取り、キャンバスへの描画や修正、ファイルの保存にそのままつなげる
- **判断は人に残す**：修正や補完は、プレビューで確認してから適用する。SAKSHI と Modeler Pro では、AI を使うかどうか自体も選べる

---

## スキルセット

**UI/UX デザイン**（20 年以上）
- UX リサーチ / 情報設計 / プロトタイピング
- デザインチームの立ち上げ・マネージメント

**AI を活用したプロダクト開発**
- AI 向けの開発計画書・チケット設計（要件・完了条件・検証値の定義）
- 実機検証とフィードバックによる品質の作り込み
- 成果物: iOS アプリ 3 本・macOS アプリ 1 本を App Store でリリース（SwiftUI / SwiftData / MapKit / CloudKit / Vision / Gemini API 等）、Figma プラグイン 6 本（Community 公開 1 本・組織内リリース 5 本）

---

## 経歴

フリーランス > Yahoo! JAPAN > コンパス（Qubena） > NAIN > Qrio > トヨタファイナンス > F&M net（現職）

---

*最終更新: 2026-11-15*
