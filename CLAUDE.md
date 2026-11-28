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
4. `質問箱・回答`（`notes`）— 「📮 質問箱」（投稿エリア）／「🗂 質問ログ」（一覧・すべて/未回答/回答済みフィルタ・解決済みトグル）／各質問カード内の回答スレッドと「💬 回答を書く」フォームに分かれている。投稿・回答は共有ボードへ自動送信し、新着は通知する（下記「7」「8」「9」）
5. `AIチューター・全単元類題`（`ai`）— **現在は保留中（Coming Soon 表示）**。`AI_TUTOR_ENABLED = false` のフラグで本体UI（`#aiFeatureArea`）を隠し、`#aiComingSoon` を表示している。再開時はフラグを `true` にするだけで元の機能に戻る（`callGemini()` 等のコードは削除しないこと）

※「円周角の定理」タブは定期テスト範囲変更（2次関数＋平行四辺形・ひし形の合同の証明）に伴い削除済み。

### 2. Canvas描画とタッチ/ポインタ操作
- Canvas描画処理、およびタッチ/ポインタによるドラッグ操作を壊さないこと。
- Canvasの `touch-action: none` 指定を削除しないこと（iPad/スマホでのドラッグ中にスクロールが発生してしまうため）。
- 座標変換（`getBoundingClientRect()` と `canvas.width / rect.width` のスケーリング）を維持すること。

### 3. XSS対策（`escapeHtml`）
- ユーザー入力・学習メモ・共有ボードから取得したデータ・AI生成テキスト・ドリル表示など、動的な文字列を `innerHTML` やテンプレートリテラルでHTML描画する際は、必ず `escapeHtml()` を適用すること。
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

### 7. 学習メモの共有と写真
- メモは各端末の `localStorage`（`math_lab_notes`）に必ず保存する。共有は共有ボード（GAS）のみで行う。手動の「共有コード（`MATHLAB1:`）」機能は**廃止済み**なので再導入しないこと。
- 外部から来たメモ（共有ボード）は `sanitizeNote()` で検証・正規化し、`uid` で重複を除外する。メモの `id` は受け取った側で振り直す（`onclick` に埋め込むため安全な整数のみ）。
- 写真は Canvas で最大 800px・JPEG 品質 0.7 に縮小し、大きければさらに段階的に縮小して保存する。
- 画像の `src` には `isSafeImageDataUrl()` を通過した `data:image/(jpeg|png|webp);base64,...` のみを使うこと（SVG・`javascript:` 等は不可）。
- `localStorage` の容量オーバーは `commitNotes()` で検知して変更を元に戻す。
- 写真はファイル選択・カメラ撮影・貼り付け（`paste` イベント／「📋 画像を貼り付け」ボタン）のすべてを `processPhotoFile()` に通す。

### 8. 共有ボード（Google Apps Script 連携）
- サーバー側コードは `index.html` 内の `<script type="text/plain" id="gasCodeTemplate">` に置いている（実行されない。設定画面の「GAS用コードをコピー」で使う）。GAS側を変えたらここも更新し、利用者に再デプロイを案内すること。この中に `</script>` を書かないこと。
- 設定（GAS WebアプリURL・ルームID）は端末の `localStorage` にのみ保存する。リポジトリにURLやIDを書き込まないこと。送信先は `isValidGasUrl()` で `https://script.google.com/.../exec` に限定する。
- 質問と回答は同じ形のデータで、`entryType`（`question` / `answer`）と `parentId`（回答先の質問の `uid`）で区別する。`parentId` が空・不正なものはすべて質問として扱う（過去のメモとの後方互換）。質問ログは端末のメモ（`notesData`）と共有ボード（`classNotes`）を `uid` でまとめて表示する（`buildQaModel()`）。
- 「📮 質問を投稿」（`addNote()`）と「💬 回答を送信」（`submitAnswer()`）は、共有URLが設定されていれば**必ずローカル保存を先に行い**、その後に自動で `syncNoteToClass()` → `gasPost()`（`Content-Type: text/plain` でCORSプリフライトを回避）で送る。ドリルの「質問箱に保存」は送らない（ローカルのみ。再送ボタンで共有できる）。
- 回答を送る前に GAS の版（`v: 2`）を確認し、古いGASには送らない（古いGASは回答を質問として保存してしまうため）。
- シートの列：A〜J は旧版と同じ、K=`entryType`、L=`parentId`、M以降が写真の分割データ。旧版の行（K列から写真）もそのまま読めるようにしている（`readRow_()`）。列の並びを変えないこと。
- 書きかけの回答は `answerDrafts` に保持し、一覧を描き直してもフォーカスとカーソル位置を戻す。日本語の変換中（`qaComposing`）は描き直さない。
- メモごとの送信状態：`⏳ 送信中`（`syncingIds`、保存しない）／`☁ 全員へ共有済`（`synced: true`）／`⚠ ローカル保存のみ（未送信・未共有）・再送`（`synced: false` または未送信）／URL未設定時は `📱 ローカル保存のみ`。
- 共有ボードは同じルームに接続した**全員（先生・生徒）**が見る。取得した投稿はメモリ上（`classNotes`）にだけ持つ。自動取得は、ページが表示中かつ質問箱画面を開いているときは6秒ごと、それ以外（別のタブ・別の画面）は新着通知のため30秒ごと（`classPollTick()`、「自動更新」OFFなら取得しない）。失敗が続いたら最大60秒まで間隔をのばす。取得中の「今すぐ更新」は終わり次第もう一度取得する。
- 差分取得（`since`＝シートの行番号）なので、授業中にシートの行を削除しないこと（シート全体の消去は検知して取り直す）。
- シートの1セルは5万文字が上限のため、写真は45,000文字ごとに分割して保存する。セルの値は先頭に `'` を付けて保存し、数式や日付として解釈されないようにしている。
- GASから受け取ったデータも `sanitizeNote()` を通し、表示には `escapeHtml()` を使う。
- Googleカレンダー連携は `https://calendar.google.com/calendar/render?action=TEMPLATE...` のリンクを開くだけ（外部SDKは使わない）。

### 9. 新着通知
- 2回目以降の取得で届いた、自分以外の質問・回答だけを新着とする（`notifyNewArrivals()`）。見ている（表示中・質問箱画面・フォーカスあり）間は未読にしない。
- 未読がある間は、タイトル先頭の `(件数) `、Canvas で作った赤いドット付きファビコン、`navigator.setAppBadge`（対応環境のみ）、タブの「新着N」で知らせ、見たら `markQaSeen()` でリセットする。
- 「🔔 通知」は Notification API（許可済みのときだけ。通知本文はプレーンテキスト）、「🔊 音」は Web Audio API の短いチャイム（外部ファイルなし）。未対応のブラウザではエラーを出さずに省略する。
- GAS 側の `NOTIFY_EMAIL` を設定すると、新着投稿時に `MailApp.sendEmail` でプレーンテキストのメールを送る（空なら送らない。`NOTIFY_MIN_INTERVAL_SEC` で間隔を制限）。リポジトリに実在のメールアドレスを書き込まないこと。

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
