# Node Weekly Issue 642 まとめ (September 24, 2026)

> 出典: [Node Weekly Issue 642](https://nodeweekly.com/issues/642)
>
> 注記: nodeweekly.com および記事のリンク先ドメインの多くは、本作業環境のネットワークプロキシにより直接アクセスが 403 (EGRESS_BLOCKED) となったため、WebSearch ツールおよび直接アクセス可能だった一次情報（Node.js 公式ドキュメント、GitHub リポジトリの README 等）を突き合わせて再構成した。号内に「(Sponsor)」表記の記事は確認できなかった（該当ページ HTML を直接取得できなかったための限界あり）。

---

## 1. 12 Node 組み込み機能が npm パッケージを置き換える（リード記事）

**原文リンク:** [12 Node built-ins that replace npm packages](https://flaviocopes.com/node-builtins/)（Flavio Copes 氏）

### ① 目的・問題意識

Node.js のコアには近年、これまで npm パッケージに頼らざるを得なかった機能（HTTP クライアント、環境変数ロード、ファイル監視、ANSI 色付け、テストランナー、深いクローン、SQLite、glob、スクリプトランナー、サンドボックス、単一実行ファイル化、WebSocket クライアント）が続々と組み込まれている。記事は「axios, nodemon, dotenv, jest, chalk, glob などを置き換える 12 の組み込み機能」を、Node 24 LTS における安定度（Stability）と、それぞれの「落とし穴（gotcha）」付きで整理したもの。結びの主張は「パッケージを戻す前に、その機能の落とし穴を読んで自分が本当にそれに当たるか考えるべきで、大半のプロジェクトは当たらない。依存を削れば、そのパッケージの更新・推移的依存ツリー・インストール時間が丸ごと消える」というもの。

以下、記事で扱われる 12 の技術それぞれを網羅する。

### 1-1. `fetch` が `axios` を置き換える

- **①目的:** 依存なしでネイティブに HTTP リクエストを行う。
- **②技術詳細:** Node 21 以降デフォルト有効、Node 24 では完全に安定（Stable）。

```javascript
try {
  const res = await fetch('...');
  if (!res.ok) {
    // レスポンス自体のエラー（5xx, 4xx）
    switch (res.status) {
      case 400: /* Handle */ break;
      case 401: /* Handle */ break;
      case 404: /* Handle */ break;
      case 500: /* Handle */ break;
    }
  }
  // ここで正常なレスポンスとして扱える
} catch (err) {
  // リクエスト自体のエラー（ネットワークエラー）
}
```

- **③なぜ効果的か:** 単純なリクエストのために依存パッケージとその推移的依存ツリーを丸ごと削減できる。ただし `fetch()` は axios と違い、404 や 500 などの非 2xx レスポンスで**例外を投げない**（ネットワーク層の失敗時のみ reject する）。`res.ok` を自分でチェックする必要があり、axios のインターセプターや自動 JSON 変換も失われるというトレードオフがある。

### 1-2. `--env-file` が `dotenv` を置き換える

- **①目的:** 依存なしでネイティブに `.env` ファイルを読み込む。
- **②技術詳細:** Node 24.10.0 で安定化（22.21.0 にもバックポート済み。それ以前の 22.x では警告が出る）。

```
# .env
PORT=3000
```

```bash
node --env-file=.env app.js
```

```javascript
console.log(process.env.PORT); // "3000"
```

複数ファイル指定（後勝ち）:

```bash
node --env-file=.env --env-file=.development.env app.js
```

存在しなくてもエラーにしない場合:

```bash
node --env-file-if-exists=.env app.js
```

プログラムからの利用（CommonJS）:

```javascript
const { loadEnvFile } = require('node:process');
loadEnvFile();
console.log(process.env.PORT); // '1234' が出力される
```

プログラムからの利用（ESM）:

```javascript
import { loadEnvFile } from 'node:process';
loadEnvFile();
console.log(process.env.PORT);
loadEnvFile('./config/.env'); // カスタムパス
```

- **③落とし穴（なぜ完全な置き換えではないか）:** 変数展開（`URL=http://${HOST}:3000` のような式）に対応しておらず `$` がそのまま文字列になる（展開が必要なら `dotenv-expand` が別途必要）。`--env-file` は `NODE_OPTIONS` をファイルから読んで適用できるが、`process.loadEnvFile()` はタイミングが遅くこれができない。引用符付き値・引用符内の複数行値・`#` コメントは両者とも対応。

### 1-3. `--watch` が `nodemon` を置き換える

- **①目的:** ネイティブなファイル監視による自動再起動。
- **②技術詳細:** Node 22.0.0（および 20.13.0）で安定化、18.11 から利用可能。

```bash
node --watch app.js
```

- **③落とし穴:** `--watch` は `--run` と併用できない（`node --watch --run dev` は期待通りに動かない）。デフォルトではエントリーファイルと import されたモジュールのみを監視し、`fs.readFile()` で読むだけの JSON やテンプレート、`.env` の変更では再起動されない。`--watch-path=./config` で監視対象を追加できるが、macOS と Windows のみサポートで Linux では import されたモジュールしか監視できない。

### 1-4. `util.styleText` が `chalk` を置き換える

- **①目的:** 依存ゼロでネイティブに ANSI テキスト装飾を行う。
- **②技術詳細:** Node 22.13.0（および 23.5.0）で安定化、20.12 で追加。

```javascript
import { styleText } from 'node:util'
console.log(styleText('green', 'Deployed'))
console.log(styleText(['bold', 'red'], 'Failed'))
```

chalk からの移行例:

```javascript
// chalk.red("Error message")  →
styleText("red", "Error message")
// chalk.red.bold("Error: Operation failed")  →
styleText(["red", "bold"], "Error: Operation failed")
```

- **③落とし穴:** `styleText` は出力ストリームが TTY かどうかを判定し、TTY でなければエスケープコードなしのプレーンテキストを返す。強制的に色コードを付けたい場合（パイプで別のカラー化処理に渡す等）は `{ validateStream: false }` を指定する。

### 1-5. `node:test` が `jest` / `mocha` を置き換える

- **①目的:** ゼロコンフィグで動くネイティブのテストランナー＋アサーションライブラリ。
- **②技術詳細:** Node 20.0.0 以降安定。`node:assert` は長らく安定。ただしカバレッジ計測は `--experimental-test-coverage` が必要（Stability 1）、モジュールモックも実験的フラグが必要、watch モード（`node --test --watch`) も Experimental のため「ローカル開発では良いが CI スクリプトには入れるべきでない」とされる。

```javascript
import assert from 'node:assert';
import test from 'node:test';

test('synchronous passing test', (t) => {
  assert.strictEqual(1, 1);
});

test('synchronous failing test', (t) => {
  assert.strictEqual(1, 2);
});

test('asynchronous passing test', async (t) => {
  assert.strictEqual(1, 1);
});
```

```bash
node --test
# または
node --test test-file.js
```

デフォルトの検出パターン: `**/*.test.{cjs,mjs,js}`, `**/*-test.{cjs,mjs,js}`, `**/test.{cjs,mjs,js}`、および `test/` 配下のファイル。失敗したテストがあると終了コード `1`。

- **③なぜ効果的か:** 外部ライブラリのインストールなしにテストが書ける。ただしカバレッジ・モジュールモック・watch モードなど周辺機能は依然実験段階であり、Jest/Mocha の全機能を置き換えるものではない。

### 1-6. `structuredClone` がディープクローン系パッケージ（`lodash.cloneDeep`, `clone` 等）を置き換える

- **①目的:** 循環参照・Date・Map・Set にも対応したネイティブなディープクローン。
- **②技術詳細:** Node 17.0.0 以降グローバルに利用可能（import 不要）。

```javascript
const clone = structuredClone(original)
```

- **③落とし穴:** 同期処理のため、メガバイト級の巨大オブジェクトではイベントループをブロックしうる。関数や DOM ノードはクローンできず `DataCloneError` を投げる。小さく平坦なオブジェクトではスプレッド構文や `Object.assign()` の方がオーバーヘッドが少ない場合がある。

### 1-7. `node:sqlite` が `better-sqlite3` などを置き換える

- **①目的:** ネイティブビルド不要・追加ドライバ不要の組み込み SQLite。
- **②技術詳細:** Stability 1.2（Release Candidate）。Node 24.15 以降で RC 化。

```javascript
import { DatabaseSync } from 'node:sqlite';

const database = new DatabaseSync(':memory:');

database.exec(`
   CREATE TABLE data(
     key INTEGER PRIMARY KEY,
     value TEXT
   ) STRICT
`);

const insert = database.prepare('INSERT INTO data (key, value) VALUES (?, ?)');
insert.run(1, 'hello');
insert.run(2, 'world');
insert.close();

const query = database.prepare('SELECT * FROM data ORDER BY key');
console.log(query.all());
// [ { key: 1, value: 'hello' }, { key: 2, value: 'world' } ]
query.close();
```

- **③落とし穴:** `node:` プレフィックスでのみ存在し `import 'sqlite'` は失敗する。`DatabaseSync` の全操作が同期実行のため、重いクエリは JS スレッドを SQLite の応答までブロックする。（実際に OWASP の cve-lite-cli リポジトリでも「better-sqlite3 を組み込みの node:sqlite に置き換える」という Issue が立っており、この移行圧力は実際に起きている。）

### 1-8. `fs.glob` / `fs.promises.glob` が `glob` / `fast-glob` を置き換える

- **①目的:** ネイティブなパターンベースのファイル検索。
- **②技術詳細:** Node 22 で実験的追加、Node 24 で安定化（安定マーカーは二次情報のため未確証）。`glob` パッケージと同じ `minimatch` 系マッチャーを内部で使用するため挙動は概ね同じだが、一部オプションでエッジケースの差異がある。
- **③落とし穴:** `fsPromises.glob` は配列ではなく**非同期イテレータ**を返すため `.map()` や `.length` がそのまま使えない点でつまずきやすい。配列が欲しい場合は同期版の `node:fs` の `globSync()` を使う。

### 1-9. `node --run` が `npm-run-all` などのスクリプトランナーを置き換える

- **①目的:** `package.json` の `"scripts"` を高速にネイティブ実行する。

```bash
node --run <script>
```

- **②技術詳細:** `npm run` との意図的な違いが 2 点ある。
  1. `pre<script>` / `post<script>` フックは実行され**ない**。
  2. `npm_package_*` のようなパッケージマネージャ由来の環境変数は自動付与され**ない**。

  Node の公式ドキュメントはこれを「最も一般的なユースケースでのトップクラスの性能に焦点を当てるための意図的な制限」と明言している。`--env-file` / `--env-file-if-exists` は `--run` と併用可能。

- **③落とし穴:** `--watch` と併用できない（1-3 参照）。pre/post スクリプトが黙ってスキップされるため、npm 流のライフサイクルフックに依存したプロジェクトでは壊れる可能性がある。

### 1-10. Permission Model がサンドボックス用ラッパーパッケージを置き換える

- **①目的:** fs・child_process・worker・アドオン・WASI・inspector へのアクセスをネイティブに制限する「シートベルト」機構。
- **②技術詳細:** `--permission` フラグで有効化。

```bash
$ node --permission index.js
Error: Access to this API has been restricted
    at node:internal/main/run_main_module:23:47 {
      code: 'ERR_ACCESS_DENIED',
      permission: 'FileSystemRead',
      resource: '/home/user/index.js'
    }
```

```bash
$ node --permission --allow-fs-read=* --allow-fs-write=* index.js
Hello world!
```

主なフラグ: `--allow-child-process`, `--allow-worker`, `--allow-net`, `--allow-addons`, `--allow-wasi`, `--allow-ffi`, `--allow-openssl-store`, `--allow-fs-read=<path>`, `--allow-fs-write=<path>`。

- **③落とし穴（公式ドキュメントの明記事項）:** 「悪意あるコードの存在下でセキュリティ保証を提供するものではない。悪意あるコードは Permission Model を回避し、制限なく任意コードを実行できる」と明記されている。付与されたパス外へのシンボリックリンクは辿られてしまう、既存のファイルディスクリプタはモデルを迂回する、`process._debugProcess()` は制限対象外、`node:fs` は制限されるが `node:sqlite` など他のモジュールは対象外、ワーカースレッドには継承されない、といった限界がある。`--permission-audit` はブロックせず違反をログに残す監査モード。

### 1-11. Single Executable Applications (SEA) が `pkg` / `nexe` を置き換える

- **①目的:** 外部パッケージャなしで Node アプリと実行時を単一のネイティブバイナリにまとめる。
- **②技術詳細:** Stability 1.1（Active development / Experimental）。

```bash
echo 'console.log(`Hello, ${process.argv[2]}!`);' > hello.js
```

`sea-config.json`（非 Windows）:

```json
{ "main": "hello.js", "output": "sea" }
```

`sea-config.json`（Windows）:

```json
{ "main": "hello.js", "output": "sea.exe" }
```

```bash
node --build-sea sea-config.json
```

署名: macOS `codesign --sign - sea` / Windows（任意）`signtool sign /fd SHA256 sea.exe`

実行:

```bash
$ ./sea world
Hello, world!
```

- **③落とし穴:** クロスプラットフォームビルド（例: darwin-arm64 上で linux-x64 の SEA をビルド）では、コードキャッシュ・スナップショットがビルドしたプラットフォームでしかロードできないため `useCodeCache: false` と `useSnapshot: false` の指定が必須で、指定しないと生成された実行ファイルが起動時にクラッシュしうる。Node 26 以降で簡易な `--build-sea` フラグが追加されたが、従来の `--experimental-sea-config` + `postject` 注入方式も依然存在する（ただし新方式に取って代わられつつある）。

### 1-12. ネイティブ `WebSocket` クライアントが `ws`（クライアント用途）を置き換える

- **①目的:** 依存なしでブラウザ互換のネイティブ WebSocket クライアントを使う。
- **②技術詳細:** Node 22.4.0 で安定化（22.0.0 からデフォルト有効）、Node 24 でも安定。

```javascript
const socket = new WebSocket('ws://localhost:8080');

socket.addEventListener('open', event => {
  console.log('WebSocket connection established!');
  socket.send('Hello Server!');
});

socket.addEventListener('message', event => {
  console.log('Message from server: ', event.data);
});

socket.addEventListener('close', event => {
  console.log('WebSocket connection closed:', event.code, event.reason);
});

socket.addEventListener('error', error => {
  console.error('WebSocket error:', error);
});
```

JSON パターン:

```javascript
const socket = new WebSocket('ws://localhost:8080');
socket.addEventListener('open', () => {
  const data = { type: 'message', content: 'Hello from Node.js!' };
  socket.send(JSON.stringify(data));
});
socket.addEventListener('message', event => {
  try {
    const receivedData = JSON.parse(event.data);
    console.log('Received JSON:', receivedData);
  } catch (error) {
    console.error('Error parsing JSON:', error);
    console.log('Received data was:', event.data);
  }
});
```

- **③落とし穴:** あくまで**クライアント専用**であり、Node には依然としてネイティブな WebSocket **サーバー**機能がない。着信接続を受けるサーバー側実装には引き続き `ws`（または `socket.io`）が必要なため、`ws` パッケージがサーバーサイドから完全に消えるわけではない。

---

## 2. Node v22.23.3（LTS パッチリリース）

**原文リンク:** [Node v22.23.3 リリース](https://github.com/nodejs/node/releases/tag/v22.23.3)

### ① 目的・問題意識

Node.js 22（コードネーム 'Jod'）の LTS ラインにおける定期パッチリリース。2026-09-23 リリース、@juanarbol が準備、@aduh95 がリリース担当。

### ② 具体的な技術詳細

**依存関係の更新:**
- ルート証明書 → NSS 3.125
- Corepack → 0.36.0
- npm → 10.9.9
- ICU → 78.3
- OpenSSL → 3.5.8
- Undici → 6.28.1
- c-ares → 1.34.8
- tzdata → 2026c

**Node-API の追加:** `napi_create_typedarray` での `SharedArrayBuffer` サポート、新関数 `napi_create_external_sharedarraybuffer`。

**主な修正:** ビルドシステムのバイナリアップロード周りの修正、ESM ローダーのファイルシステムパッチ可能性の復旧、シリアライズ不能な URL に対する URL セッター処理、ソースマップに依存しない関数名保持、dgram のリンクローカルスコープテスト修正、HTTP/2 の RST_STREAM use-after-free の堅牢化。

**セキュリティに関する注記（要検証）:** HTTP/2 領域の著名な CVE として **CVE-2026-56848**（`src/node_http2.cc` の `Http2Stream::SubmitRstStream()` におけるヒープ use-after-free、CWE-416、CVSS 7.5、HackerOne 経由で "hahahkim" 氏が報告、Matteo Collina 氏が修正）が存在する。複数の独立ソース（HeroDevs、Vulners/Red Hat、BleepingComputer、warp2search）はこの CVE が **22.23.2**（他系統では 26.5.1、24.18.1）で修正されたとしている。一部の検索結果は 22.23.3 のコミット一覧にも「HTTP/2 use-after-free」修正を含めており、これが 22.23.2 の履歴の重複表示なのか、追加の堅牢化コミットなのかを一次情報（GitHub の raw diff）で確証できなかった。技術的な根本原因は、拒否されたストリームに対して Node が `RST_STREAM` を送信する際、`nghttp2` がまだ `nghttp2_session_mem_recv()` の処理中であるにもかかわらず再入的に `nghttp2_session_mem_send()` を呼び出してしまい、受信処理がまだ参照しているストリーム状態を解放してしまう点にある。修正はストリームクローズメッセージの処理を受信ステップ完了後まで遅延させるというもの。

### ③ なぜそれが効果的か

依存ライブラリの CVE 対応版への追随と、HTTP/2 の重大なメモリ安全性バグ（use-after-free）の修正により、本番運用中の LTS 環境のセキュリティと安定性を維持する。ただし本リリースでの HTTP/2 修正の正確な位置づけは公開情報だけでは断定できず、実運用前にリリース差分の一次確認が推奨される。

---

## 3. Node 26.10.0（Current ライン、2026-09-22 リリース）

**原文リンク:** [Node.js Blog: v26.10.0 リリース](https://nodejs.org/en/blog/release/v26.10.0)

### ① 目的・問題意識

Current ライン（26.x）の機能追加リリース。多くの開発者が個別に `node_modules` へバンドルしていたユーティリティ（デバウンス・スロットル等）を標準ライブラリに取り込むなど、実用上の摩擦を減らす方向性。

### ② 具体的な技術詳細

**新 API:**
- `util.debounce(fn, delay, [options])` / `util.throttle(fn, interval, [options])` — 「ほとんどの人がすでに node_modules に自前で持ち込んでいたヘルパー」
- `crypto.parsePKCS12()` — PKCS#12 証明書バンドルのパース
- `fs.openAsBlobSync()` — ファイル裏付けの `Blob` への同期アクセス
- `net.BoundSocket` — ワーカースレッド・子プロセスへ転送可能に
- `perf_hooks.SlidingWindowHistogram` — 時間窓ベースの新しいヒストグラム型。既存の `Histogram` にも QRDE（quantile-relative data error）解析対応が追加
- `util.markPromiseAsHandled(promise)` — Promise を「処理済み」とマークし unhandled rejection 警告を抑制
- FFI がマウントされた仮想ファイルシステム（VFS）からライブラリをロード可能に（コンテナ化・バンドルされたネイティブモジュールのシナリオで有用）
- `node:sqlite` が JS の `undefined` を SQL の `NULL` に正しくバインドするよう修正

**主なバグ修正:** リクエスト完了後の HTTP ソケット破棄タイミング、HTTP/2 のデストロイ時のライトコールバック処理、`AbortSignal.any()` の伝播、`FileHandle.readableWebStream` の改善、DTLS/QUIC の各種修正、暗号プロバイダの最適化。ある情報源によれば本リリースには 200 件超のコミットが含まれる。

### ③ なぜそれが効果的か

よく使われるユーティリティ関数（デバウンス／スロットル）の標準搭載は、1-4〜1-12 で見た「組み込み機能が npm パッケージを置き換える」流れの延長線上にあり、依存パッケージの削減に直結する。

---

## 4. Intel Mac が Node 27 で公式ビルド対象外に

**原文リンク:** [PR #65427: build: downgrade Intel macOS support to experimental](https://github.com/nodejs/node/pull/65427)

### ① 目的・問題意識

Node.js の macOS ユニバーサルバイナリ（x86_64 + arm64）は Apple の Rosetta 2 を使ってビルドされているが、Rosetta 2 は「Node.js 27 のライフサイクル中に終了する可能性が高い」と判断された。これを受け、Antoine du Hamel (aduh95) 氏が Intel Mac サポートを公式サポートから実験的（experimental）サポートへ格下げする PR を提出した（2026-08-20 オープン、2026-09-23 マージ）。

### ② 具体的な技術詳細

- **適用開始:** Node.js 27.0.0-alpha.0 から。メジャーバージョンの引き上げを要する破壊的変更として扱われる。
- **メンテナーの合意:** Stewart X Addison (sxa) 氏が「直近の Release WG コールで議論し、反対意見はなかった」とコメント。
- **Intel Mac ユーザーの選択肢:** (1) unofficial-builds プロジェクトによる非公式ビルド（保証なし）、(2) Linux への切り替え、(3) Docker/Podman コンテナの利用。
- **文脈:** Rust・Python・Node.js が同時期に同様の Intel Mac サポート終了を進めている、業界全体の移行の一部として位置づけられている。

### ③ なぜそれが効果的か

Rosetta 2 という外部依存の終了リスクを見越して事前に対応することで、突然のビルド不能化を避け、開発リソースを arm64（Apple Silicon）ネイティブ環境に集中できる。

---

## 5. CVE Lite CLI（OWASP Lab Project）

**原文リンク:** [GitHub: OWASP/cve-lite-cli](https://github.com/OWASP/cve-lite-cli)

### ① 目的・問題意識

`npm audit` は脆弱性の CVE ID を教えてくれるが、実際に「何のコマンドを打てば直るか」までは踏み込んでくれないことが多く、特に推移的依存（transitive dependency）の脆弱性では、どの直接依存（親パッケージ）を上げれば解決するかの判断が難しい。CVE Lite CLI は、プロジェクトのロックファイルをローカルでスキャンし（コードが外部に送信されない）、OSV と npm レジストリのアドバイザリ API と突き合わせ、**そのままコピペして実行できる修正コマンド**を出力する OWASP Lab Project のツール。

### ② 具体的な技術詳細

**特徴:** npm・pnpm・Yarn・Bun の 4 大パッケージマネージャに対応。直接依存 vs 推移的依存を区別し、推移的依存の脆弱性については修正バージョンに満たない形で固定している親パッケージを更新するよう案内する「親パッケージ考慮型の改善提案」を行う。SPDX 2.3 / CycloneDX 形式の SBOM を脆弱性情報付きで生成可能。GitHub Code Scanning 向けの SARIF 出力にも対応。アカウントや API キー、設定ファイルは不要。

**インストール:**

```bash
npm install -g cve-lite-cli
# またはワンショットで:
npx cve-lite-cli /path/to/project
```

**主なコマンド:**

```bash
cve-lite /path/to/project                   # 基本スキャン
cve-lite /path/to/project --verbose         # 依存パス付きの完全な修正プラン
cve-lite /path/to/project --fix             # 検証済みの直接修正を適用し再スキャン
cve-lite /path/to/project --fail-on high    # high 以上の深刻度で非ゼロ終了
cve-lite /path/to/project --json            # JSON 出力
cve-lite /path/to/project --sarif           # GitHub Code Scanning 形式
cve-lite /path/to/project --sbom spdx       # SPDX 2.3 SBOM 出力
cve-lite /path/to/project --report          # インタラクティブな HTML ダッシュボード
cve-lite /path/to/project --offline         # ランタイム API 呼び出しなしでスキャン
```

**`--fix` の動作:** 検出したパッケージマネージャに応じた更新コマンド（`npm install` / `pnpm add` / `yarn add` / `bun add`）を提案し、提案バージョンが実際に CVE を解決するかを OSV に対して検証、推移的依存の脆弱性では強制的な直接インストールではなく親パッケージの更新を推奨、修正後に再スキャンして解決を確認する。

**`--json` 出力に含まれる情報:** アドバイザリのメタデータ（CVE/GHSA ID、深刻度、CVSS スコア）、影響を受けるパッケージと現在のバージョン、直接／推移的の分類、修正コマンドの提案、推移的パッケージへの依存パスなど。

**その他の機能:** 提案する修正バージョンがパッケージマネージャの設定した安全ウィンドウより新しい場合に警告する「リリースクールダウン考慮」、npm/pnpm/Yarn/Bun のオーバーライド設定を 13 種類の問題（孤立したピン留め、浮動タグ、脆弱なフロア等）について監査する機能、メンテナンスリスクの検出。

**差別化ポイント:** 4 大 JS パッケージマネージャすべてのロックファイルスキャン、親パッケージを考慮した推移的依存の改善提案、完全オフラインでのアドバイザリ対応、実用的な修正コマンド提示を、アカウント登録や外部ダッシュボードなしで兼ね備えている点。

なお、CVE Lite CLI リポジトリ自身にも「better-sqlite3 を組み込みの node:sqlite に置き換える」という Issue（#1237）が立っており、記事1の技術内容が実プロジェクトでも実践されていることが伺える。

### ③ なぜそれが効果的か

「脆弱性の存在を知らせる」だけでなく「どう直すか」まで自動化することで、依存関係のセキュリティ対応にかかる調査コストを大きく削減する。ローカル・オフラインでも動作するため、CI や機密性の高い環境でも利用しやすい。

---

## 6. pnpm 12.6

**原文リンク:** [GitHub: pnpm v12.6.0 リリースノート](https://github.com/pnpm/pnpm/releases/tag/v12.6.0)

### ① 目的・問題意識

モノレポ／ワークスペース運用における依存関係の重複、プロジェクトディレクトリ移動時の `node_modules` 破損、`package.json` の可読性、型パッケージの管理の煩雑さといった実務上の課題に対応するマイナーリリース。

### ② 具体的な技術詳細

**自動依存関係重複排除（`autoDedupe`）:** インストール中に互換性のある依存バージョンを自動的に重複排除する。ある依存が複数バージョンで存在し、そのうち 1 つが全ての範囲指定を満たす場合、ワークスペース全体でそのバージョンに統一する。

```yaml
# pnpm-workspace.yaml
autoDedupe: true
```

コマンド単位でも有効化可能: `pnpm install --auto-dedupe` / `pnpm add --auto-dedupe`。`--frozen-lockfile` を使ったインストールではロックファイルは変更されない（dedupe は実行されない）。

**再配置可能な `node_modules`:** macOS / Linux では、プロジェクトごと `node_modules` を移動・コピーしても `pnpm install` / `pnpm run` / `pnpm exec` がそのまま再利用できるようになった。移動後の最初のコマンド実行時にツリーをチェックし新しい場所を記録するため、`node_modules/.bin` のシムも動作し続ける。

**`package.yaml` マニフェスト対応:** `package.json` に代わる新しい YAML 形式。`pnpm add` / `pnpm update` / `pnpm remove` / `pnpm pkg` / `pnpm link` / `pnpm set-script` / `pnpm version` から編集可能で、既存のコメントとキー順序を保持する。

**`--save-types` フラグ:** `pnpm add --save-types` で、利用可能な `@types/*` パッケージを対象パッケージと一緒に `devDependencies` に保存する。TypeScript の型を同梱しているパッケージは自動的にスキップされる。デフォルトで有効化する設定:

```yaml
saveTypes: true
```

**catalog プロトコルの拡張:** catalog エントリが `file:` および `link:` プロトコルに対応し、パスは `pnpm-workspace.yaml` のディレクトリからの相対パスとして解決される。

**`pnpm tasks status`:** 並行性グループごとの実行中・待機中タスクを優先度ベースのスロット割り当てとともに一覧表示。

**`pnpm cache prune`:** 現在の pnpm バージョンで読み込めなくなったレジストリメタデータキャッシュディレクトリを削除。`--dry-run` でプレビュー可能。

**パフォーマンス:** オーバーライドの収束が多いワークスペースにおいて、`pnpm dedupe` とインストールの両方でオーバーライドチェックの並行化により高速化。

**セキュリティ／堅牢化の修正:** Cygwin/MSYS2/WSL2 における POSIX バイナリシムがシステムの `cygpath` / `wslpath` を利用するように、非推奨警告が通知文を全文漏らさないように、`.npmrc` に埋め込まれた環境変数由来の認証情報に対して設定警告を出すように、それぞれ修正。

### ③ なぜそれが効果的か

`autoDedupe` は依存ツリーの肥大化を自動抑制し、node_modules の再配置対応はモノレポの CI キャッシュ戦略やコンテナ間でのプロジェクト移動を容易にする。`--save-types` は TypeScript プロジェクトでの型パッケージの入れ忘れを防ぐ。

---

## 7. Linear、AI コーディングエージェント時代の CI 再構築

**原文リンク:** [Linear reworks CI as coding agents make validation the expensive part](https://runtimewire.com/article/linear-reworks-ci-ai-coding-verification-bottleneck)（一次情報: Linear 社エンジニアリングブログ "AI coding has made CI a bottleneck, so we reworked ours to keep up", 2026-09-21 公開、共同創業者/CTO Tuomas Artman 氏とエンジニア Mufeez Amjad 氏による）

### ① 目的・問題意識

AI コーディングエージェントはコード（とテスト）を従来より遥かに速く生成できるようになった結果、コードレビューやコード生成ではなく**CI／検証パイプラインそのものがボトルネックになる**という現象が起きている。Linear では 2026 年初頭からテストスイートがほぼ 4 倍に増加し、AI エージェントが週あたり約 2,000 件のテストを生成する規模に至った。

### ② 具体的な技術詳細（施策と効果測定）

1. **GitHub Actions から高速なサードパーティランナーへ移行:** より高速な CPU・ストレージ・キャッシュにより、ジョブが平均 **34% 高速化**。
2. **TypeScript のネイティブコンパイラ `tsgo` へ切り替え:** 週次の型チェック時間の中央値を **73% 削減**。
3. **カスタム ESLint ルールの書き換え:** TypeScript の型グラフ全体をロードせず構文のみを解析するように変更し、API のリント時間を **68%**、リポジトリ全体のリント時間を **55%** 削減。将来的な Oxlint への移行も見据えた対応。
4. **クリティカルパス上のジョブの最小化／重複セットアップの削減:** スコープを絞ったインストール、共有ベースイメージ、スキーマスナップショット、短時間チェックのバッチ化。具体的には、独立していた 7 個のチェックジョブを並行タスク実行を伴う 2 個にまとめ、月間約 **87,000 ランナー分（全 CI 使用量の 11.8%）** を削減。
5. **Vitest のシャーディング再設計:** 4 シャードから **8 シャード**へ。大きなファイルの分割とシャード数の倍増により、クリティカルジョブが初期ベンチマークで約 **19% 高速化・19% コスト削減**。最も遅いシャードは 1 週間のチューニングを経て **5.25 分 → 4.33 分**に改善。
6. **対象ファイルでのテスト分離無効化（`isolate: false`）:** 「安全」と判断したファイルではモジュール状態をテスト間で共有し、エンティティ・GraphQL・デコレータグラフの冗長な再構築を排除。これにより 1 回の実行あたりのシャード合計実行時間が **32.8 分 → 22 分**に短縮し、単独で月間約 **17% のコスト削減**を実現。この高速モードはオプトインとし、必要なティアダウンルールを設けた上で、フェイクタイマーや壊れやすい共有状態に依存するテストは分離ワーカーに残した。
7. **シャーディング改善の前提としてのセットアップ時間最適化:** シャードあたりのセットアップ時間が 110〜140 秒あった段階で単純に 8 シャードへ拡張すると、セットアップだけで 15〜19 分のランナー時間（実際のテスト実行時間を上回る）を消費してしまうため、まずシャードあたりのセットアップ時間を約 40 秒まで削減し、それにより 8 シャード化が正味の改善となった。
8. **社内の「エージェントスキル」（AI コーディングエージェントが参照する指示・設定）の更新:** AI が生成するテストが、人間のレビューでの検出に頼らず、新しいパフォーマンス制約（適切な分離設定、シャーディングに適した構造など）をデフォルトで満たすように変更。

**総合指標:** テストが約 4 倍に増える一方で、テスト 1 件あたりの CI ランナー時間はおおよそ**半減**。プルリクエストの待ち時間は 6 分超から**5 分強**に短縮。

**記事の主張の要旨:** AI エージェント主導の開発ではコード（およびテスト）生成の速度が検証インフラの処理能力を上回るため、CI のアーキテクチャ・ランナーの経済性・テスト分離戦略が中核的なエンジニアリング上の制約になりつつある。関連する dev.to の記事は「Linear はテストが 4 倍になる中で CI ランナー時間を半減させたが、レビューゲートは同じ芸当ができない」と評しており、人間によるコードレビューは AI 駆動のスループット向上に CI と同じようにはスケールしないことを示唆している。

### ③ なぜそれが効果的か

コンパイラ・リンターのアルゴリズム的なボトルネック（型グラフ全体のロード）を狙い撃ちで解消し、セットアップの重複排除によってシャーディングの並列化効果を初めて正味プラスにできた点が鍵。さらに「エージェントが生成するテスト自体をパフォーマンス制約に適合させる」という上流側の対策により、対症療法ではなく再発防止まで踏み込んでいる。

---

## 8. 「シニアエンジニアの死のスパイラル」（バーンアウト論考）

**原文リンク:** [the senior engineer death spiral](https://sunilpai.dev/posts/the-senior-engineer-death-spiral/)（Sunil Pai 氏）

> 注記: Node Weekly 側の元記事タイトルとの完全な一致は一次ページを直接取得できず確認できていないが、時期（Hacker News・TLDR.tech 2026-09-21 号・daily.dev で同時期に流通）とテーマの一致度から、これが該当記事である可能性が高いと判断した。

### ① 目的・問題意識

昇進・転職・大きなプロジェクトの担当など、役割の変化直後にエンジニアが陥りがちな失敗パターンを描いた論考。

### ② 具体的な内容

**典型的なパターン:** 役割変化に過剰反応したエンジニアが、野心的すぎる取り組みに一人で没頭し始め、進捗報告は「順調です」という曖昧なものだけになり、助けを求めたり途中経過を見せたりせず、秘密主義と働きすぎのスパイラルに陥る。

**内的な語り（引用・要約）:** 「最近あまり何も出せていない。もっと頑張らないと。十分に頑張ればいい。これは隠しておけば誰にも分からない。来週で 1 ヶ月分の仕事をこなせば誰にも気づかれない。」

**帰結:** 睡眠不足、抑うつ、食事の欠如、職場・私生活双方での人間関係の悪化。最悪の場合、燃え尽き、休職、PIP（業績改善プログラム）、解雇、あるいは自主退職に至る。

**処方箋:** 「独力での英雄的行為」で自分を証明しようとする衝動に抗うこと。意図的にスコープを一段階「下げる」こと。チームメイトにとって目に見える形で小さく具体的なタスクを手伝う存在になること。最終成果を一度に大きく見せる「結果ベース」の思考から、小さく着実に進捗を見せる「モメンタムベース」の思考へ転換すること。単発の大きな（誰にも見えない）努力ではなく、日々の観察可能な仕事を通じて信頼と信頼性を再構築すること。

### ③ なぜそれが効果的か

「見えない努力の蓄積で一発逆転を狙う」という直感的だが危険な戦略の心理的メカニズムを言語化し、代わりに小さく可視的な進捗を積み重ねる行動指針を示すことで、チーム内の信頼を損なわずに立て直す具体的な道筋を提供している。

---

## 9. HTML 要素の周期表（ビジュアル/デモ記事）

**原文リンク:** [Periodic Table of HTML Elements](https://blog.alena.rocks/en/artifacts/html-elements/)（Alena Batitskaia 氏）

### ① 目的・問題意識

Josh Duck 氏によるオリジナルの "Periodic Table of the HTML5 Elements"（Brad Frost 氏の著書 *Atomic Design* でも取り上げられた著名なビジュアライゼーション）を、現行の HTML 仕様に合わせて刷新したもの。HTML5 要素の全体像とカテゴリ分類を、仕様を線形に読むのではなく一目で把握できるようにする教育・リファレンスツール。

### ② ビジュアル・デモ内容の言語化

現行 HTML 仕様の**全 115 要素**を、化学の周期表そっくりの格子状レイアウトに配置し、各要素は化学元素のセル（短いタグ記号＋名前）のように表示される。セルは **カテゴリ／グループごとに色分け**されており、グループは化学ではなく HTML 上の構造的役割に対応する：**Root（ルート）、Metadata（メタデータ）、Sections（セクション）、Grouping（グルーピング）、Text-level / inline（インラインテキストレベル）、Edits（編集）、Embedded（埋め込み）、Tabular（表形式データ）、Forms（フォーム）、Interactive（インタラクティブ）、Scripting（スクリプティング）、void 要素**。このレイアウトにより、開発者は本物の周期表で族や周期を眺めるのと同じ感覚で、あるカテゴリの要素群を一目で見渡せる。**検索機能**、**仕様セクション（＝上記グループ）によるフィルタ**、各要素から **MDN の該当ドキュメントへの直リンク**といったインタラクティブ機能を備える。

### ③ なぜそれが効果的か

HTML の全要素を一望できる視覚的な参照モデルは、仕様書を線形に読むよりも要素同士の関係性・分類を直感的に理解させる。検索・フィルタ・MDN への直リンクにより、参照ツールとしての実用性も兼ね備えている。

---

## 10. secure-eval-worker: Node.js における最小権限 JavaScript 実行（「安全な eval()」）

**原文リンク:** [Introducing secure-eval-worker](https://blog.platformatic.dev/introducing-secure-eval-worker) / [GitHub: platformatic/secure-eval-worker](https://github.com/platformatic/secure-eval-worker)

### ① 目的・問題意識

`eval()`（あるいはそれをワーカースレッドに移しただけのもの）は、自分で権限を剥奪しない限り、ホスト側の全権限（ファイルシステム・ネットワーク・環境変数・プロセス制御）をそのまま継承してしまう。`secure-eval-worker` は、信頼できないコード・AI 生成コード・プラグインの JavaScript（スクリプト、ES モジュール、あるいは信頼できるローカルファイル）を実行する際に、**最小権限での実行をデフォルト**にすることを目的としたライブラリ。以下の要素を組み合わせている。

1. `node:worker_threads` によるワーカー
2. ゲストコードのロード前に権限を剥奪した状態で有効化される Node の **Permission Model**
3. ホスト側が明示的に定義する狭い範囲の**ケイパビリティ関数**（アンビエントなシステムアクセスの代わりにゲストへ公開する API）
4. ホストとゲスト間の**認証付きメッセージプロトコル**（セッションごとの HMAC 鍵＋単調増加するシーケンス番号）
5. ワーカーのライフサイクル全段階における**リソース上限**（ソースサイズ、メッセージサイズ、呼び出し回数、メモリ）

README には「これは多層防御（defense-in-depth）であり、完全なサンドボックスではない」と明記されている。本当に敵対的なマルチテナントのワークロードには、依然として OS レベルのプロセス／コンテナ分離（CPU・メモリ・fs・ネットワーク・syscall の制限）が必要である。理由は、複数のワーカーが単一の V8 プロセスを共有しているため、意味論的なバグやネイティブなメモリ安全性の欠陥が起きればエスケープしうること、リソース上限が `ArrayBuffer` / WebAssembly / ネイティブ確保を制約しないこと、タイムアウトを超えても割り込み不能なネイティブ処理は継続すること、abort シグナルを無視するホスト関数はそのまま動き続けてしまうことなどによる。

### ② 具体的な技術詳細・コード例（GitHub README より）

インストール:

```bash
npm install secure-eval-worker
```

**ワンショット実行:**

```javascript
import { runUntrustedCode } from 'secure-eval-worker'

const value = await runUntrustedCode(
  'return input.values.reduce((sum, value) => sum + value, 0)',
  {
    input: { values: [10, 20, 12] },
    timeoutMs: 500,
    environment: { LANG: 'C' }
  }
)
console.log(value) // 42
```

（ソースは `input` を一つ受け取る非同期関数の本体として実行され、データは `workerData` を経由し、結果は認証済みのプライベートチャネルで返る。）

**永続セッション（スクリプト形式）＋ホストケイパビリティ関数:**

```javascript
import { createUntrustedWorker } from 'secure-eval-worker'

const component = createUntrustedWorker(`
  const profile = await users.find(input.userId)
  send({ status: 'started', profile })
  onMessage(async (message) => {
    send({ observed: message })
    return audit.record(message)
  })
`, {
  input: { userId: 'user-123' },
  hostFunctions: { users: { find: async (id) => database.users.find(id) } },
  lifetimeTimeoutMs: 60_000
})

component.on('message', (message) => console.log('component message:', message))
await component.ready
component.postMessage({ kind: 'notification' })
const reply = await component.request({ kind: 'question' })
await component.terminate()
```

**永続セッション（ES モジュール形式）:**

```javascript
const component = createUntrustedWorker(`
  export default async function setup ({ input, send, onMessage }) {
    send({ status: 'started', input })
    onMessage(async (message) => ({ echo: message }))
  }
`, {
  type: 'module',
  input: { tenant: 'example' },
  lifetimeTimeoutMs: 60_000
})

await component.ready
console.log(await component.request('hello')) // { echo: 'hello' }
await component.terminate()
```

**ローカルファイルの実行:**

```javascript
import { runUntrustedFile } from 'secure-eval-worker'

const result = await runUntrustedFile('./components/calculate.mjs', {
  rootDirectory: './components',
  input: { values: [10, 20, 12] },
  timeoutMs: 500
})
```

```javascript
import { createUntrustedWorkerFromFile } from 'secure-eval-worker'

const component = await createUntrustedWorkerFromFile(
  './components/service.mjs',
  {
    rootDirectory: './components',
    input: { tenant: 'example' },
    lifetimeTimeoutMs: 60_000
  }
)
```

**ホストケイパビリティ関数パターン:**

```javascript
const component = createUntrustedWorker(`
  onMessage(async (id) => records.find(id))
`, {
  hostFunctions: {
    records: {
      find: async (id) => {
        // ここで信頼できない引数の検証・認可を行う
        return database.records.find(id)
      }
    }
  }
})
```

ルール: ホスト関数は名前空間化され、読み取り専用のグローバルとしてインストールされる。同期・非同期いずれのホスト関数もゲスト側には常に Promise として返る。予期しないホストエラーは秘匿され、メッセージとコードが開示して安全と分かっている場合のみ `HostFunctionError` として投げられる。

**ホスト関数内でのコンテキスト取得:**

```javascript
import { getHostFunctionContext } from 'secure-eval-worker'

const context = getHostFunctionContext()
// 戻り値: abortSignal, sessionId, requestId, requestIndex, hostFunctionName
```

**リソース上限の設定（ワンショット）:**

```javascript
const value = await runUntrustedCode(source, {
  input: { /* データ */ },
  timeoutMs: 1000,
  maxSourceBytes: 65536,
  environment: { LANG: 'C' },
  resourceLimits: {
    maxOldGenerationSizeMb: 128,
    maxYoungGenerationSizeMb: 10,
    codeRangeSizeMb: 10,
    stackSizeMb: 4
  },
  signal: abortController.signal,
  language: 'javascript', // または 'typescript'
  hostFunctions: { /* ... */ },
  diagnostics: true
})
```

**リソース上限の設定（永続セッション）:** デフォルトでは同時受理ワーカー数 **4**（キューなしで即座に失敗）、最大ソースサイズ **64 KiB**、最大セットアップ入力／プロトコルメッセージ **1 MiB**、非請求メッセージ総数上限 **1,024 件**（累計 **16 MiB** 上限）、ホスト関数呼び出し総数 **256 件**（同時実行上限 **32**）。

```javascript
const component = createUntrustedWorker(source, {
  input: { /* ... */ },
  type: 'script', // または 'module'
  language: 'javascript',
  startupTimeoutMs: 1000,
  messageTimeoutMs: 1000,
  lifetimeTimeoutMs: 30000,
  maxHostFunctionCalls: 256,
  maxInFlightHostFunctions: 32,
  maxInputBytes: 1048576,
  maxMessageBytes: 1048576,
  maxOutputMessages: 1024,
  maxOutputBytes: 16777216,
  diagnostics: {
    maxRecords: 100,
    maxBytes: 65536,
    maxRecordBytes: 4096
  }
})
```

**ワーカー受理制御:**

```javascript
import { configureWorkerAdmission } from 'secure-eval-worker'
configureWorkerAdmission({ maxConcurrentWorkers: 4 })
```

容量超過時は永続セッションで `ERR_UNTRUSTED_WORKER_CAPACITY`、ワンショットで `ERR_UNTRUSTED_CODE_CAPACITY` を送出。

**エラーハンドリング:**

```javascript
try {
  const result = await runUntrustedCode('throw new Error("Guest error")')
} catch (error) {
  console.log(error.code)        // 機械可読なコード
  console.log(error.remoteStack) // ゲスト側のスタックトレース（信頼できない診断情報 — 認可判断には絶対使わないこと）
}
```

**TypeScript（型消去のみ、型チェックなし）:**

```javascript
const result = await runUntrustedCode<OutputType, InputType>(
  'return input.value as number',
  {
    language: 'typescript',
    input: { value: '42' }
  }
)
```

**再利用可能なランナーファクトリ:**

```javascript
import { createRunner } from 'secure-eval-worker'

const runner = createRunner({
  timeoutMs: 5000,
  environment: { LANG: 'C' }
})

const result1 = await runner(source1, { input: data1 })
const result2 = await runner(source2, { input: data2, signal: mySignal })
```

**永続セッションのインスタンス API:**

```javascript
component.ready              // セットアップ完了後に解決する Promise
component.postMessage(value) // 送りっぱなしの配信
component.request(value)     // 送信してハンドラの応答を待つ
component.terminate()        // 冪等な終了リクエスト
component.closed             // ワーカー終了時に解決する Promise
component.on('message', (value) => {})
component.on('diagnostic', (record) => {})
component.on('error', (error) => {})
component.on('exit', () => {})
```

### ③ なぜそれが効果的か

- 全てのワーカーは **Permission Model を有効化した状態**で起動する。ソース文字列ワーカーは `worker` 権限のみ、ローカルファイルワーカーはそれに加えプライベートなスナップショットへの読み取り専用 fs アクセスのみを持つ。さらに**すべてのワーカーは、ゲストコードを評価する直前に `worker` 権限自体を剥奪**するため、ゲストコードが無制限な子ワーカーを自分で生成することもできない。ブートストラップ処理では Node の組み込みオブジェクトのアイデンティティをレビュー済みのマニフェストと照合し、fs 書き込み（ステージ済みスナップショットを除く）・ネットワーク・子プロセス起動・ネイティブアドオンロード・inspector・WASI を無効化する。ファイルハンドルにも上限がある（ローカルワーカーごとに 64、プロセス全体で 256）。
- **データプロトコルは厳格**で、プリミティブ型、列挙可能プロパティのみを持つプレーンなオブジェクト／配列、`ArrayBuffer`、非共有の TypedArray、`Date`、`RegExp`、`Map`、`Set` のみ転送可能。`SharedArrayBuffer`、プラットフォームオブジェクト（Blob・ポート・ファイルハンドル・暗号鍵）、カスタムクラスインスタンス、アクセサ、シンボル、非列挙プロパティはすべて**拒否**される。これにより、リッチなオブジェクト参照を信頼境界を越えて漏らすタイプのサンドボックス脱出経路の多くを塞いでいる。
- **要件:** Node.js **26.5.1 以上**（Permission Model と関連セキュリティ修正のため）。ホスト自体が `--permission` 下で動作している場合は `--allow-worker` が必要。ファイルパスベースの実行にはホストの読み取り権限と一時ディレクトリへのアクセスが必要。
- **ライセンス:** MIT。

### ④ デモ・ビジュアル要素

なし（ライブラリの README であり、視覚的なデモは含まれない）。

---

## まとめ

| トピック | 重要度 | アクション |
|---------|--------|-----------|
| 12 の Node 組み込み機能が npm パッケージを置き換え | ★★★ | 新規・既存プロジェクトで axios/dotenv/nodemon/chalk/jest 等の依存を段階的にネイティブ機能へ置き換え、各機能の「落とし穴」を確認した上で採用する |
| Node v22.23.3 | ★★☆ | LTS 環境は速やかに追随。HTTP/2 use-after-free 修正の正確な適用範囲は一次情報で要確認 |
| Node 26.10.0 | ★★☆ | `util.debounce`/`throttle` など新規ユーティリティの採用を検討 |
| Intel Mac が Node 27 で experimental 化 | ★★☆ | Intel Mac の CI/開発環境がある場合、Linux または Docker への移行を早めに計画 |
| CVE Lite CLI (OWASP) | ★★★ | CI に組み込み、依存の脆弱性検出と修正コマンド提示を自動化する |
| pnpm 12.6 | ★★☆ | モノレポでは `autoDedupe` と `--save-types` の有効化を検討 |
| Linear の CI 再構築 | ★★★ | AI エージェントによるテスト急増に備え、コンパイラ切り替え・リント高速化・シャーディング再設計・テスト分離の見直しを参考にする |
| シニアエンジニアの死のスパイラル | ★★☆ | 大きな役割変化の直後は、結果ベースでなくモメンタムベースで小さく進捗を可視化する |
| HTML 要素の周期表 | ★☆☆ | HTML5 要素の分類学習・リファレンスとしてブックマーク |
| secure-eval-worker | ★★★ | 信頼できない／AI 生成コードを実行する必要がある場合、`eval()` や素のワーカースレッドの代わりに検討（ただし完全なサンドボックスではない点に注意） |
