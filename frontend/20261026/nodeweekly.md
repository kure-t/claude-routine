# Node Weekly Issue #644（2026-10-15）まとめ

- 号タイトル: The NodeConf EU 2026 talks are online
- 取得方法: nodeweekly.com への直接アクセスは不可のため WebSearch の要約と二次情報で構成。検索時点（2026-10-26）で最新は #644（編集者は翌週休刊、次号は 10/29 予定）。
- **注意**: 号の記事一覧全文・各リンク先の全文は取得できませんでした（WebFetch は名前解決失敗）。コード全文など未確認の項目は「未取得」と明記しています。推測での補完はしていません。

## 1. NodeConf EU 2026 のトークが YouTube で公開
- 内容: Node の新しい FFI 機能、Emscripten / WebAssembly、V8 のヒープ管理、パフォーマンスプロファイリング等のトーク。NodeConf EU はイタリア（ボローニャ、9/29-30）で開催。
- 詳細（各トークのコード・デモ）: 動画リンクのためスキップ。

## 2. Deno が Cloudflare に参画（celld を workerd に統合）
- 目的・背景: Cloudflare Workers のセルフホストを容易にする。Durable Objects が workerd では単一インスタンスでしか動かない点が課題。Deno の celld と workerd を統合して解決を狙う。
- 発表: 2026-10-09、Ryan Dahl と Kenton Varda（Cloudflare）の共同ブログ。買収か否かは報道で差があり、金額は非公表。
- 影響: Deno ランタイムは今後 1 年はバグ修正・セキュリティ更新のみ（その後コミュニティ維持を期待）。Deno Deploy は 6 か月で終了し有料ユーザーは Cloudflare Workers へ移行。JSR は継続（基盤は Cloudflare へ移行）。
- コード: なし。

## 3. Node.js v26.12.0 (Current) リリース（10/14）／24.22 も言及
Node 24.22 の詳細は未取得。v26.12.0 の主な変更（公式リリースノートより。API の使用例コードは同ページになく未取得）:
- **SEMVER-MINOR**
  - WebAssembly / RX サポートが無い場合は例外を投げる (#66533)。
  - `http` に `websocketMask()` / `websocketUnmask()` を追加 (#66335)。WebSocket フレームのマスク処理を標準で扱えるようにし、ws 等のサードパーティ実装への依存や自前実装を減らす目的。
- **Config file**: 修正と stable 化 (#66431)。permission フラグが子プロセスへ伝播 (#66559)。
- **Streams**: `stream/iter` に `dump()`/`dumpSync()`、`pipeToSync()` の `failOnIncompleteClose` オプション、エラー処理・バックプレッシャー修正多数。
- **Process**: `process.finalization` が release candidate (#66436)。
- **Module**: ほとんどの ES module を同期ロード (#62530)。package map 付き import の `require` を修正 (#66565)。
- **Permissions**: `--allow-fs-vfs` 追加 (#66554)。
- **Crypto**: ルート証明書ストアを Environment ごとに保持 (#66411)。
- **SQLite**: `enableDefensive()` が再オープン後も維持、`aggregate()` の引数検証、巨大文字列は abort せず例外化。
- **Test runner**: 失敗する `todo` テストをスイート内で再実行、`t.assert` にモック呼び出しアサーション。
- **依存更新**: acorn 8.19.0, sqlite 3.54.0, simdjson 5.0.3。Python 3.15 対応。
- 補足: 10/21 に v26.12.1 が公開済み。

## 4. WebSocket Servers in JavaScript: Node.js, Bun and Deno in 2026（Daniel Lemire, 2026-10-10）
- 目的・問題意識: Node.js / Bun / Deno の WebSocket サーバの往復（ping-pong）スループットとレイテンシを比較。
- 結果: Node.js 26 + uWebSockets.js が約 49,000 round trips/秒、Deno 約 40,000、Bun 約 37,000。Node Weekly は「C 実装に近い低レイテンシで Bun・Deno を上回る」と紹介。
- 関連: Bun 側で v1.4.2→v1.4.3 で ping-pong スループットが約 45% 低下したとの Issue (oven-sh/bun #46968) が、このベンチマークを使って報告されている。
- 効果的な理由: uWebSockets.js は C++ 実装でメッセージ処理のオーバーヘッドが小さい。ベンチコード全文・計測条件は未取得（ブログ本文に到達できず）。

## 未取得・要再確認
- #644 に含まれるその他の記事（ライブラリ・チュートリアル枠、"sponsor" 表記の除外判定を含む）。
- 上記各記事のコード全文。
- 次回は 10/29 の #645 公開後に再実行を推奨。

Sources: https://nodeweekly.com/issues/644 / https://lemire.me/blog/2026/10/10/websocket-servers-in-javascript-node-js-bun-and-deno-in-2026/ / https://nodejs.org/en/blog/release/v26.12.0
