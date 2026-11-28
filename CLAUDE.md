# CLAUDE.md — math-app プロジェクトルール

## アプリ概要

中学生（主に中3・定期テスト対策）向けの数学学習Webアプリ。**単一ファイル `index.html` のみ**で構成され、ビルド工程はない（GitHub Pages 等でそのまま配信）。

| 項目 | 使用技術 |
| --- | --- |
| マークアップ | HTML5 |
| ロジック | Vanilla JavaScript（フレームワーク・バンドラーなし） |
| スタイル | Tailwind CSS（CDN読み込み） |
| 図形・グラフ描画 | HTML5 Canvas |
| データ保存 | `localStorage`（学習メモ、Gemini APIキー、最新版チェック用SHA など） |
| AI機能 | Gemini API をブラウザから直接呼び出し（APIキーは利用者が各自入力し、端末の `localStorage` にのみ保存） |

新しいファイルやビルドツール・npm依存を追加せず、単一ファイル構成を維持すること。

## 保持すべき既存機能と制約

### 1. 5つの主要タブ
以下のタブの動作とレイアウトを維持すること（`switchTab()` と `xxxModule` のID対応を壊さない）。

1. `2次関数・動点`（`funcs`）
2. `放物線と面積（苦手克服）`（`parabolaArea`）
3. `平行四辺形・ひし形 証明`（`proof`）— 図形＆性質シミュレーター（Canvas）と、通信不要の合同証明演習ドリル（`PROOF_DRILLS` 配列にデータを追加すれば問題を増やせる）
4. `学習メモ`（`notes`）
5. `AIチューター・全単元類題`（`ai`）

※「円周角の定理」タブは定期テスト範囲変更（2次関数＋平行四辺形・ひし形の合同の証明）に伴い削除済み。

### 2. Canvas描画とタッチ/ポインタ操作
- Canvas描画処理、およびタッチ/ポインタによるドラッグ操作を壊さないこと。
- Canvasの `touch-action: none` 指定を削除しないこと（iPad/スマホでのドラッグ中にスクロールが発生してしまうため）。
- 座標変換（`getBoundingClientRect()` と `canvas.width / rect.width` のスケーリング）を維持すること。

### 3. XSS対策（`escapeHtml`）
- ユーザー入力・学習メモ・AI生成テキスト・ドリル表示など、動的な文字列を `innerHTML` やテンプレートリテラルでHTML描画する際は、必ず `escapeHtml()` を適用すること。
- プレーンテキストで十分な箇所は `textContent` を使う。
- XSS脆弱性を新たに作らないこと。

### 4. レスポンシブ表示
- モバイル / iPad / PC のいずれでもレイアウトが崩れないこと（Tailwind のブレークポイント `md:` / `lg:` 等と `max-w-*` + `w-full` のCanvasサイズ指定を維持）。

### 5. Gemini API のモデルフォールバック・リトライ
`callGemini()` の以下の順序を維持すること。

1. `gemini-3.8-flash`
2. `gemini-3.8-flash`（待機後にリトライ）
3. `gemini-3.5-flash`
4. `gemini-3.5-flash-lite`

APIキー不正エラー時は即座に中断し、それ以外のエラーは次のモデルへフォールバックする。Gemini APIキーの設定・保存・削除UI（APIキー設定モーダル）は使いやすく維持すること。

### 6. 公開サイトの更新通知
- `checkRemoteUpdate()` は公開リポジトリ `trhtiachijp/math-app` の最新コミットを認証なしで確認し、更新があれば更新バナー（`#updateBanner`）を表示して再読み込みする。
- 通信エラー・レート制限などはサイレントに無視し、既存機能を妨げないこと。
- ブラウザから GitHub へ書き込む機能（Contents API への PUT、PAT の入力・保存）は**廃止済み**。再導入しないこと。

## Git運用ルール

サイトの更新はローカル（Claude Code）から `git push` で行う。

1. 変更前後に、シークレット（APIキー、GitHubトークン、パスワード等）がコードやコミット履歴に混入していないことを確認する。
   - 例：`git diff` を目視し、`AIza`、`ghp_`、`github_pat_`、`sk-` などのパターンが含まれていないか検索する。
2. 編集後、`index.html` の構文崩れ（タグ・括弧の対応）、未定義関数・存在しない要素IDの参照がないことを確認する。
3. 以下まで実行して反映する。
   ```bash
   git add <変更したファイル>
   git commit -m "<変更内容が分かるメッセージ>"
   git push origin main
   ```
4. `index.html` は CRLF 改行で管理されている。差分が全行にならないよう改行コードを維持すること。
