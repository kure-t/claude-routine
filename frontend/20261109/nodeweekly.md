# Node Weekly Issue #646 (2026-11-05) まとめ

- 対象: https://nodeweekly.com/issues/646 「Node 27's first alpha brings breaking changes」
- 最新判定: WebSearch で確認できた最大番号が #646（#645 は 2026-10-29）
- 注意: 実行環境から nodeweekly.com 等への直接アクセスは名前解決エラーとなるため、WebSearch の要約と、取得できた nodejs.org の公式リリースノートを元に作成。nodeweekly.com 本文の全項目は確認できていない。
- 除外: "sponsor" 記事（Postgres.js 4.0 は sponsored 扱いのため詳細省略）

---

## 1. Node 27 の最初のアルファ版（v27.0.0-alpha.0.0.0, 2026-10-29）

出典: https://nodejs.org/en/blog/release/v27.0.0-alpha.0.0.0

### 概要・目的
新しいリリース体制で、各メジャーは約6か月アルファ期間を経て安定版になる。v27 の安定版は 2027年4月、LTS 入りは 2027年10月、EOL は 2030年4月頃。アルファは main ブランチから作られ、ライブラリ作者の互換性テスト向けで本番利用は想定されない。

### 1-1. V8 14.6 → 15.6.75.13
- ①目的: 最新の JS 言語機能とエンジン改善の取り込み。
- ②詳細（デフォルト有効）: `Math.sumPrecise()`, `Intl.Locale.prototype.variants`, `Iterator.prototype.join()` / `includes()`, `Iterator.zip()` / `Iterator.zipKeyed()`。
  実験的（フラグ必須）:

  | 機能 | フラグ |
  |---|---|
  | `import defer` | `--js-defer-import-eval` |
  | Immutable `ArrayBuffer` | `--js-immutable-arraybuffer` |
  | RegExp buffer boundaries | `--js-regexp-buffer-boundaries` |

  挙動変更: Math が LLVM の libm を使い最終ビットが変わりうる／`--random-seed` 指定時の `Math.random()` 列が変わる／`error.stack` がプロトタイプチェーンを辿らない／`await` のマイクロタスク数減少で順序が変わりうる／大きな配列の `sort()` は PowerSort（安定）／ヒープスナップショットは内部オブジェクト・数値・全ソースを含み、約2倍の時間とサイズ。WebAssembly: asm.js は Wasm にコンパイルされなくなり、`--experimental-wasm-<name>` は `--wasm-<name>` に改名。
- ③効果: 新機能で記述が簡潔になる（例: `Iterator.zip` で複数イテレータの結合が標準で可能）。内部挙動の変更は互換性テストが必要な箇所の目印になる。

### 1-2. デフォルトヒープ上限の変更
- ①問題意識: 固定的な既定値だとコンテナ／大容量マシンでメモリを活かせない、または不足する。
- ②詳細: `--max-old-space-size` 未指定時、上限は「物理メモリの半分、256MB〜4GB にクランプ」。`Buffer` / `ArrayBuffer` のメモリもヒープ上限に算入される。8GB マシンでは old generation が 2GB → 4GB。従来の想定に依存するなら明示指定する:
  ```sh
  node --max-old-space-size=4096 app.js
  ```
- ③効果: 環境に応じた自動調整。ただし Buffer 算入により、Buffer を多用するアプリは OOM の条件が変わるため確認が必要。

### 1-3. シリアライズ形式の変更（互換性破壊）
- ②詳細: `v8.serialize()` / `structuredClone()` / `postMessage()` が新フォーマットになり、古い Node では読めない。`v8.cachedDataVersionTag()` とコンパイルキャッシュは V8 のパッチごとに変わり、他パッチのキャッシュは拒否されうる。
- ③影響: 複数バージョン混在環境（ワーカー、キャッシュ共有）では不整合に注意。アップグレード後はコンパイルキャッシュの再生成を。

### 1-4. 破壊的変更（SEMVER-MAJOR）の主なもの
- ビルド/プラットフォーム: OpenSSL 3 以上必須、同梱 FIPS provider 削除、Intel macOS は experimental に降格、`NODE_MODULE_VERSION` が 153（ネイティブアドオンは再ビルド）、`node::ObjectWrap` 非推奨。
- Crypto: OpenSSL 4.1.0、legacy コードと engine サポート削除、エンコーディング検証の厳格化、Web Crypto で resizable backing store を拒否、未初期化 DH の `computeSecret` は throw、`Hmac.digest()` の複数回呼び出しは runtime deprecation。
- ネットワーク: `net` の複数回 `listen()` は throw／`dns` 逆引きで hosts ファイルを参照しない／`dgram` はリテラル IP で custom `lookup` を使わない／`http` の `keepAliveTimeout` 既定が 65 秒、`req.headers` が null プロトタイプ、`new` なし呼び出しは runtime deprecation／`http2` の既定ウィンドウサイズ増加／`_tls_*` は EOL。
- コアAPI: `domain` は runtime deprecation（`AsyncLocalStorage` ベースに移植）、旧 `AsyncLocalStorage` 実装削除、`--no-experimental-websocket` 削除、`fs.glob` をネイティブ実装、`Blob` は `SharedArrayBuffer` を拒否、REPL は inspector 使用。
- 権限: 新フラグ `--allow-env`。
- 依存: npm 12.1.0、V8 15.6.75.13。

### 1-5. 追加機能（semver-minor）と主なパッチ
- `fs.mkstemp()`, `zlib.crc32c()`, `process` の finalization が unregister 関数を返す, `test_runner` の `mockReturnValue` とプロミス用ヘルパー, `--experimental-strip-private-modules`, VFS のフラグ解除, Alpine を Tier 2 に（最低 Alpine 3.21）, AbortSignal 高速化。
- パッチ: CVE-2026-58042（`dns` の大きな `resolveAny` 応答）修正、TLS の受信レコード結合、SQLite の busy/locked バックアップ再試行と結果行の shared-shape 化（高速化）、`stream/iter` の最適化。
  （公式ノートにシグネチャ付きの完全なコード例は無く、正確な API は各ドキュメントで要確認。）

### 1-6. アップグレード時のチェックリスト
1. メモリ前提があるなら `--max-old-space-size` を明示
2. ネイティブアドオン再ビルド（ABI 153）
3. serialize/structuredClone/postMessage データの旧版非互換に注意
4. アップグレード後にコンパイルキャッシュを再生成
5. ソースビルドは OpenSSL 3+ 必須
6. `http` keep-alive 65 秒と `dns` 逆引き挙動の確認
7. `domain`、`_http_*`、`_tls_*` を使わない

---

## 2. npm 12 が同梱（Node 27 アルファは npm 12.1.0）

出典: Node Weekly の要約、GitHub Changelog「Upcoming breaking changes for npm v12」(https://github.blog/changelog/2026-06-24-upcoming-breaking-changes-for-npm-v12/) ほか二次情報。npm 12 は 2026-07-08 に GA。
- ①目的: インストール時のサプライチェーン攻撃対策。暗黙の挙動を明示的な opt-in に変更。
- ②詳細: インストールスクリプト（preinstall/install/postinstall）は既定で実行されない（`allowScripts` 既定オフ）／`--allow-git` 既定 `none`／`--allow-remote` 既定 `none`（リモートURL依存は明示許可が必要）／`npm view --json` は常に配列を返す。
- ③効果: 悪意あるパッケージの自動コード実行や、予期しない git/URL 依存の取り込みを防ぐ。移行は npm 11.16.0 以上で警告を確認するのが推奨。CI でネイティブビルド等に scripts が必要な場合は許可設定が必要。
  （公式リリースノートは直接確認できず、二次情報に基づく。）

---

## 3. 新リリースのパッケージ

（以下は WebSearch で得た Node Weekly の要約のみ。リンク先の本文・コードは取得できず未確認。）

### 3-1. ora 10
- ①目的: ターミナルのスピナー表示。
- ②詳細: Node 22 が必須に。自身が描画したフレームだけを消去し、プログラムの他の出力を壊さない。
- ③効果: 並行ログ出力とスピナーの表示崩れが減る。

### 3-2. Got 16.2
- ①目的: 人に優しい HTTP リクエストライブラリ。
- ②詳細: `.json()` が Standard Schema 準拠のバリデータ（Zod、Valibot、ArkType など）を受け取れる。コード例は未取得。想定イメージ（推測、公式例ではない）:
  ```js
  import got from 'got';
  import { z } from 'zod';
  const User = z.object({ id: z.number(), name: z.string() });
  const user = await got('https://api.example.com/user/1').json(User);
  ```
- ③効果: 検証と型付けを1回の呼び出しで行える。

### 3-3. express-fileupload 1.6
- ①問題: Content-Type チェックの ReDoS。
- ②詳細: セミコロンが長く連続するヘッダでイベントループが停止しうる脆弱性を修正。
- ③対応: 利用中なら 1.6 へ更新。

### 3-4. その他
- Postgres.js 4.0: sponsored のため対象外。
- 号内にその他のセキュリティ更新があるとの記載があるが、個別内容は未確認。

---

## 未確認事項
- nodeweekly.com 本文の全項目一覧、動画リンクの有無、各リンク先のコード全文。ドメイン許可後に再取得すると補完できる。
