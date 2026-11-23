# Node Weekly Issue #647 まとめ (November 12, 2026)

> 出典: [Issue #647: 22 Node, Bun and Deno stacks benchmarked — Node Weekly](https://nodeweekly.com/issues/647)
>
> **取得方法についての注記:** nodeweekly.com への直接アクセスは不可（このセッションでは名前解決もできない）だったため、WebSearch で号を特定した。検索で確認できた最新号は #647（2026-11-12）。#648（11/19 号と思われる）は検索インデックスに見つからなかった。各記事の詳細は、取得できたリンク先（GitHub / nodejs.org）を WebFetch で読み、取得できなかったリンク先（blog.x.com、netlify.com、dev.to、jasnell.me）は検索結果から再構成した。
> 「sponsor」と付いた記事は除外している。

---

## 目次

1. 22 の Node / Bun / Deno スタックのベンチマーク（巻頭記事）
2. Node に実験的な「グリーンスレッド」（`node:green_threads`, PR #67288）
3. Node.js 2026年11月11日セキュリティリリース（16 CVE）
4. X が Web アプリを Fastify ベースで作り直した
5. Web Worker API が Node の main ブランチでデフォルト有効化（PR #65323）
6. 組み込み YAML モジュールの提案（`node:yaml`, PR #67157 / Issue #67347）
7. Netlify Server が Early Access に
8. Postgres.js 4.1（4.1.0 / 4.1.1、4.0 の要点も）

---

## 1. 22 の Node / Bun / Deno スタックのベンチマーク（巻頭記事）

**元記事:** Node Weekly #647 の巻頭記事。原典は「js-runtimes-benchmark」と呼ばれるプロジェクトで、解説記事は [FastAPI vs Node.js vs Bun: One PostgreSQL Setting Was Worth Up to 31% (DEV Community)](https://dev.to/rudratosh/fastapi-vs-nodejs-vs-bun-one-postgresql-setting-was-worth-up-to-31-no-runtime-changed-4b1h)。どちらも直接は読めず、内容は検索結果から再構成した。

### ① 目的・問題意識

ランタイムの比較ベンチマークの多くは "Hello World" や JSON を返すだけのマイクロベンチマークで、実際のアプリの性能とはずれる。この記事では、**同じ Postgres バックエンドの API アプリ（RealWorld 仕様の "Conduit" アプリ）** を、Node・Bun・Deno の 3 ランタイムとそれぞれの人気フレームワークを組み合わせた **22 スタック** で実装した。そのうえで、データベース専用マシンに対して負荷をかけて比較している。

### ② 技術詳細

- **構成:** 22 の「ランタイム × フレームワーク」スタック（Express / Fastify / Hono / Elysia など、各ランタイムの代表的なフレームワーク）。どれも同じ RealWorld API を実装する。
- **DB:** 専用マシン上の PostgreSQL。
- **主な結論（Node Weekly の要約）:** **フレームワークの選択よりもランタイムの選択のほうが結果を大きく左右した。**
- **解説記事による重要な指摘:** ランタイムとフレームワークを固定したまま、**Node.js から PostgreSQL への接続の仕方を変えるだけで最大 31% の差**が出た。
  - Bun の `Bun.SQL` と、Deno スタックで使われた `postgres.js` は、**プリペアドステートメントを自動で使う**。
  - Node の既定のドライバ構成（例: `pg` を素のテキストクエリで使う）では自動で prepare しないため、そのぶん遅い。
  - つまり「ランタイム差」と見えた差の一部は、**ドライバの既定値の違い**だった。

設定の違いを示すコード例（記事の主張を説明するためにこちらで書いた例。原文のコードではない）:

```js
// pg (node-postgres): name を指定するとプリペアドステートメントとして
// サーバ側で一度だけパースされ、以降は再利用される
import pg from 'pg';
const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });

// 毎回パース・プランニングされる（既定の使い方）
await pool.query('SELECT * FROM articles WHERE slug = $1', [slug]);

// 名前付き = prepared statement として再利用される
await pool.query({
  name: 'article-by-slug',
  text: 'SELECT * FROM articles WHERE slug = $1',
  values: [slug],
});
```

```js
// postgres.js: 既定で prepare: true（自動でプリペアドステートメントを使う）
import postgres from 'postgres';
const sql = postgres(process.env.DATABASE_URL /* , { prepare: true } */);
const [article] = await sql`SELECT * FROM articles WHERE slug = ${slug}`;
```

### ③ なぜ重要か

- 実アプリに近い条件（DB 往復あり）で測ると、ランタイム間の差はマイクロベンチより大幅に縮む。他の 2026 年の記事でも、Bun の優位は合成ベンチの約 2 倍から 25〜30% 程度に縮むと報告されている。
- それでも差が残る場合は、**ドライバの既定値（プリペアドステートメント、パイプライン化）**のほうが、フレームワークの選択より効くことがある。Node を使う場合は、ランタイムを乗り換える前にドライバ設定を見直すほうが効果的。
- 注意: 22 スタック全体の数値表は確認できなかった。数字を引用する場合は原典を確認すること。

---

## 2. Node に実験的な「グリーンスレッド」（`node:green_threads`）

**元記事:** [src: experimental green threads implementation by jasnell · PR #67288](https://github.com/nodejs/node/pull/67288) / [Green threads for Node.js (James Snell のブログ)](http://www.jasnell.me/posts/green-threads-in-node)

### ① 目的・問題意識

Node で JS コードを隔離して並行に動かす手段は、これまで主に 2 つあった。

| 手段 | 問題 |
|---|---|
| `worker_threads` | 1 Worker ごとに **OS スレッド 1 本 + イベントループ + Node API 一式**を持つため重い。1000 個で約 6.7〜7.9GB。 |
| `vm` コンテキスト | 安いが、**同じ isolate を共有するのでセキュリティ境界にならない**。暴走コードを止めるには呼び出しごとに `timeout` を付ける必要があり、スループットが大きく落ちる。 |

マルチテナントのプラグイン実行、ユーザーのスクリプト実行、エッジ関数のような用途では、**強く隔離されていて、軽量で、CPU やメモリを制限できる**実行単位が欲しい。

### ② 技術詳細

- 新モジュール `node:green_threads`（実験的）。PR ブランチをビルドし、`--experimental-green-threads` フラグを付けて使う。
- **各グリーンスレッドは独立した V8 isolate**（`vm` より強い境界）だが、**Node.js の API（`fs` など）は一切持たない**。使える機能はホストが明示的に渡す。
- 多数のグリーンスレッドが**少数の OS スレッドのプールを共有する**（既定は 8 本、`NODE_GREEN_THREADS_POOL_SIZE` で変更できる）。
- 仕事は**メッセージ、RPC 呼び出し、モジュールの exports**の 3 経路で渡す。1 回ごとに独立した呼び出しコンテキストで実行される。
- ホストはスレッドごとに **CPU 時間の上限と メモリの上限** を設定でき、`terminate()` で止められる。
- 後のコミットで追加されたもの: スタブ付き RPC、ホストが渡す `resolve()` / `load()` フックによる ES モジュール対応、`MessageChannel` / `MessagePort`、許可リストで制限された `BroadcastChannel`、`close()`、`setStallPolicy()`、`setUnresponsiveTimeouts()`、`--allow-green-threads` パーミッションフラグ。
- 予定（この PR には含まれない）: `TextEncoder` / `TextDecoder`、`URL` / `URLSearchParams`、`structuredClone()`。
- 制約: `Atomics.wait()` はグリーンスレッド内では例外を投げる。値は structured clone でやり取りするので、オブジェクトを本当に共有して書き換えることはできない。
- PR 本文によると、コードの大半は AI が書き、手作業で調整したもの。mcollina が承認済み（2026-11-14 に ready for review）。

#### 基本例（PR 本文より）

```js
const { GreenThread } = require('node:green_threads');

const greenThread = new GreenThread(`
  onmessage = ({ data }) => postMessage(data * 2);
`);
greenThread.on('message', (value) => {
  console.log(value);  // Prints: 42
  greenThread.terminate();
});
greenThread.postMessage(21);
```

#### プラグインホストの例（PR 本文より）

```js
const plugin = new GreenThread(source, {
  type: 'module',
  filename: `file:///plugins/${name}/index.js`,
  modules: plugins.shared,
  env: {
    async config(key) { return plugins.config[key]; },
    log(...args) { console.log(`[${name}]`, ...args); },
  },
  resourceLimits: { maxTaskCpuTimeMs: 100, maxMemoryMb: 32 },
});
const { transform } = await plugin.exports;
console.log(await transform('hello world'));
```

- `env` に渡した関数（`config`、`log`）だけが、プラグインから見えるホスト機能になる（capability 方式）。
- `resourceLimits` により、1 タスクあたり CPU 100ms、メモリ 32MB を超えると強制的に止まる。
- `plugin.exports` で ES モジュールの export を RPC として非同期に呼べる。

#### ベンチマーク（PR 本文。2026-11-14 の更新コメントより）

**アイドル状態のインスタンス 1000 個:**

| 種類 | 起動数/秒 | 1 インスタンスあたり PSS | PSS 合計 | OS スレッド |
|---|---|---|---|---|
| グリーンスレッド | 3,739 | 1.36 MB | 1,364 MB | 8 |
| グリーンスレッド（ウォームプール） | 80,778 | 1.38 MB | 1,377 MB | 8 |
| Worker | 977 | 6.17 MB | 6,171 MB | 1,000 |

ポインタ圧縮ビルドで V8 の既定 isolate グループ、または共有 `IsolateGroup` を使うと 0.95 MB/個、ウォームプールなら 101,977 起動/秒。

**テナント 1000 個・リクエスト 50,000 件:** グリーンスレッド 201,834 req/s・PSS 1,415MB、Worker 157,475 req/s・PSS 6,216MB。

**メッセージ往復:** グリーンスレッド 49,187/s、Worker 49,221/s（どちらも約 20.3µs）で同等。

**CPU バウンド（`fib(27)` × 2000 回）:**

| 構成 | タスク/秒 |
|---|---|
| グリーンスレッド 8 | 5,023 |
| Worker 8 | 5,052 |
| グリーンスレッド 88（`NODE_GREEN_THREADS_POOL_SIZE=88`） | 21,516 |
| Worker 88 | 22,230 |
| タスクごとにグリーンスレッド 1 個（2000 個、プール 8） | 4,151 |

**100 テナント中 1 つが無限ループした場合:**

| | 止めた仕組み | 停止までの時間 | 200ms 間に他テナントへ応答した数 |
|---|---|---|---|
| グリーンスレッド | `maxTaskCpuTimeMs: 50` | 52 ms | 46,332 |
| Worker | 50ms タイマーの後に `terminate()` | 65 ms | 36,135 |

**`vm` プールとの比較（devsnek の提案で追加計測）:**

| 構成 | req/s | p50 | p99 |
|---|---|---|---|
| グリーンスレッド | 186,481 | 0.32 ms | 1.27 ms |
| vm プール（Worker 8） | 190,804 | 0.33 ms | 0.72 ms |
| vm プール（Worker 8）+ `timeout` | 80,415 | 0.63 ms | 2.59 ms |
| vm プール（Worker 88）+ `timeout` | 58,469 | 1.00 ms | 2.90 ms |

- 1 テナントがマイクロタスクの無限ループに入った場合: グリーンスレッドは 54ms で停止し、失うのはそのテナント 1 つだけ。`vm` プールは 101ms 後に Worker ごと終了し、**13 テナントを巻き添えにする**。
- 1 テナントがヒープを使い切った場合: グリーンスレッドは 1 テナントを失い 99 テナントが応答を続ける。`vm` プールは 13 テナントを失う。
- プラグインホストの例: `spin`（無限ループ）は CPU 時間 100ms で停止。`hog`（`ArrayBuffer` を確保し続ける）は 32MB の上限で確保に失敗する。Worker では 100ms のタイムアウト内に 91MB まで膨らんだ。

### ③ なぜ効果的か

- **isolate レベルの隔離**を保ちながら、OS スレッドとイベントループを共有するので、メモリは Worker の約 1/4〜1/6。起動もウォームプールなら数万〜10 万回/秒と桁違いに速い。
- **CPU 時間とメモリの上限が isolate 単位で強制される**ので、暴走したテナントだけを確実に止められ、他のテナントの遅延（最遅 3ms）を守れる。`vm` + `timeout` のようにスループットを犠牲にする必要がない。
- Node API を既定で持たない **capability 方式**なので、信頼できないコードの実行に向く。
- 弱点: プールが既定で 8 本なので、重い並列計算は Worker（またはプールを大きくすること）のほうが向いている。

---

## 3. Node.js 2026年11月11日セキュリティリリース

**元記事:** [Node.js — Wednesday, November 11, 2026 Security Releases](https://nodejs.org/en/blog/vulnerability/november-2026-security-releases)

### ① 目的・問題意識

HTTP/2・TLS・HTTPS・パーミッションモデルを中心に、**16 件の CVE**（最高深刻度は High）が修正された。多くはリモートから認証なしで攻撃でき、プロセスのクラッシュや**ヒープメモリの漏洩（TLS 秘密鍵を含むおそれ）**につながる。

### ② 修正版と CVE 一覧

**修正版:** 22.x → **v22.23.5** / 24.x → **v24.22.3** / 26.x → **v26.13.1**
（Node 20 と 25 は EOL のため修正版は出ないが、影響は受ける）

| CVE | 深刻度 | 内容 | 影響ライン |
|---|---|---|---|
| CVE-2026-123863 | High | **HTTP/2 リモートアボート:** `~Http2Session()` 内の `CHECK_EQ(current_nghttp2_memory_, 0)` が失敗する。セッションより長生きしたストリームが切り離されず、認証なしのクライアントが TLS でも平文でもプロセスを落とせる（CVE-2024-27983 の亜種）。 | 24, 26 |
| CVE-2026-123864 | High | **HTTPS ヒープ漏洩:** 長さ 0 の TLS write で `llhttp_execute()` が再入され、use-after-free と範囲外読み取りが起きる。ヒープ（TLS 秘密鍵を含みうる）がレスポンスボディに混入する。平文の `node:http` は影響なし。 | 22, 24, 26 |
| CVE-2026-123865 | High | **HTTP/2 リソースリーク:** 接続終了中にプロトコルエラーが起きるとセッションが残り続け、メモリと FD を抱え込む。`GOAWAY(NO_ERROR)` で誘発できる（CVE-2026-48937 の修正漏れ）。 | 22, 24, 26 |
| CVE-2026-123866 | High | **HTTP/2 破棄時の use-after-free:** バックプレッシャーで書き込み待ちのストリームを破棄すると、解放済みや無関係のヒープが相手に送られる。`RST_STREAM` やストリームごとのタイムアウトで誘発できる。 | 22, 24, 26 |
| CVE-2026-126051 | High | **TLS サーバのクラッシュ:** 遅延した SNI / OCSP / セッションのコールバックがソケット破棄後に走り、`'error'` イベントではなく捕捉されない例外になる。認証なしでリモートから誘発できる。 | 24, 26 |
| CVE-2026-123872 | High | **TLS 1.2 クライアント証明書のバイパス:** 再ネゴシエーション中に servername を変え、既存の認証状態のまま、より厳しいクライアント証明書ポリシーを持つホストへ移れる。 | 22, 24, 26 |
| CVE-2026-123874 | High | **HTTP/2 use-after-free:** `respondWithFile()` / `respondWithFD()` で、write がキューに残ったままリセットされたストリームが解放され、完了時に解放済みオブジェクト経由の仮想関数呼び出しが起きる。 | 24, 26 |
| CVE-2026-123828 | Medium | **HTTP/2 ヒープオーバーフロー:** `session.altsvc()` が origin 長で確保したバッファを 1 バイト越えて書く。長い正当な `:authority` で誘発できる。 | 24, 26 |
| CVE-2026-123867 | Medium | **mTLS 認可のバイパス:** ホストごとの CA だけを持ちサーバ証明書を持たない SNI コンテキストが、黙って既定のクライアント CA にフォールバックする。 | 22, 24, 26 |
| CVE-2026-123868 | Medium | **HTTP/2 メモリ枯渇:** 読み出されていないストリームに溜まったデータが `maxSessionMemory` に計上されず、上限として機能しない。 | 24, 26 |
| CVE-2026-123871 | Medium | **認証情報の漏洩:** `ERR_PROXY_INVALID_CONFIG` のメッセージに、ユーザー名とパスワードを含む生のプロキシ URL が載る（CVE-2026-48615 の修正が 4 経路を見落としていた）。 | 22, 24, 26 |
| CVE-2026-123877 | Medium | **証明書検証のバイパス:** `https.Agent` で `ca` を省略したリクエストと `ca: []` のリクエストのセッションキャッシュキーが同じになり、既定のトラストストアで認証したセッションが厳しいポリシー側で再利用される。 | 22, 24, 26 |
| CVE-2026-123875 | Medium | **HTTP/2 クラッシュ:** `respondWithFile()` のダウンロードをキャンセルすると `StreamPipe` の破棄順序が誤り、NULL 参照になる。数回のキャンセルで落ちる。 | 22, 24, 26 |
| CVE-2026-123829 | Low | **パーミッションモデルのバイパス:** macOS / Windows で再帰的な `fs.watch` が、拒否されたサブディレクトリ内のファイル名を漏らす（中身は漏れない）。 | 22, 24, 26 |
| CVE-2026-123862 | Low | **zlib のクラッシュ:** ストリーミング中に `deflate.params()` の後で `destroy()` を呼ぶと、捕捉できない `ERR_INTERNAL_ASSERTION` が起きる。 | 22, 24, 26 |
| CVE-2026-106071 | Low | **パーミッションモデルの過剰許可:** ルート直下のワイルドカード `--allow-fs-read` / `--allow-fs-write` が、より短いパスにもマッチする。例: `--allow-fs-read=/etc/passwd.*` で `/etc/passwd` も読める。 | 22, 24, 26 |

### 暫定的な緩和策

**公式アドバイザリには回避策はなく、アップグレードが唯一の完全な対策。** 以下は各脆弱性の発生条件から推測した緩和策で、十分である保証はない。

```js
// CVE-2026-123828: 攻撃者が制御できる :authority をそのまま altsvc() に渡さない
const ALLOWED = new Set(['example.com', 'www.example.com']);
const origin = req.headers[':authority'];
if (ALLOWED.has(origin)) session.altsvc(altValue, origin);
```

- CVE-2026-123864: HTTPS サーバで、リクエストボディを読み終える前に `res.flushHeaders()` を呼ばない。
- CVE-2026-123877: CA ポリシーごとに別の `https.Agent` を使う。
- CVE-2026-123867: すべての SNI コンテキストにサーバ証明書を設定する。
- CVE-2026-106071: ルート直下のワイルドカード許可を、明示的なパス指定に置き換える。
- CVE-2026-123871: ログに出た可能性のあるプロキシ URL の認証情報はローテーションする。

### ③ なぜ重要か

HTTP/2 と TLS を公開しているサーバでは、**認証なしでリモートから DoS や情報漏洩を起こせる**脆弱性が複数ある。特に CVE-2026-123864 は秘密鍵の漏洩につながりうる。22 / 24 / 26 の利用者は即時の更新が推奨され、EOL の 20 / 25 は移行が必須。

---

## 4. X が Web アプリを Fastify ベースで作り直した

**元記事:** [Rebuilding the global town square from scratch (X Engineering Blog)](https://blog.x.com/en/engineering/rebuilding-the-global-town-square-from-scratch)（直接は読めず、検索結果と二次記事 [pasqualepillitteri.it](https://pasqualepillitteri.it/en/news/32834/x-rebuilds-web-app-first-post-2-7-seconds) から再構成した）

### ① 目的・問題意識

旧 X Web は、**通常ユーザー向け、検索クローラー向け、リンクプレビュー向けの 3 つのサーバ**に分かれていた。遅い回線ではプロフィールの最初のポストが表示されるまで **22 秒以上**（X の計測では 23.9 秒）かかっていた。

### ② 技術詳細

- **スタック:** Vite + React + **Fastify** + **Platformatic Watt** + Relay（GraphQL）+ XDS（新しいコンポーネントライブラリ）。
- **サーバの統合:** 3 つのサーバを 1 つにまとめた。
- **Fastify の役割:** ミドルウェアから最終的な HTML ストリームまで、すべてのリクエストを処理する。社内パッケージ `@x-clients/server` が、セキュリティヘッダ、Cookie、メトリクス、ストリーミングレンダリング、グレースフルシャットダウンを設定から組み立てる。
- **Watt の役割:** **Node.js v26** 上で、1 プロセス内の複数の worker thread としてサーバを動かす。**イベントループの遅延やメモリが上限を超え続けた worker を入れ替える**ので、詰まったレンダリングはその worker のリクエストにしか影響しない。
- **ストリーミング SSR:** React の Streaming + Suspense を使う。先にページのシェルを送り、各セクションはデータが届いた順にストリームで流し込む。サーバで受け取った GraphQL レスポンスはページに一度だけ埋め込まれ、ブラウザ側は API を再クエリせずに再利用する。
- **開発体制:** 5 人のエンジニアが AI コーディングエージェントを使い、最初のバージョンを 6 か月弱で作った（アーキテクチャの判断と生成コードのレビューは人間が担当）。
- **ロールアウト:** ログアウト状態のページとクローラー向けはすでに新アプリで動いている。ログインユーザーは国ごとに段階的に移行中で、最初はインドネシア。

### ③ 効果

- 遅い 4G で最初のポストが表示されるまでの時間: **22 秒超 → 約 2.7 秒**（X 計測のスロー 4G、3 回の中央値）。報道では約 8 倍、絞った回線では 8〜13 倍と紹介されている。
- シェルを先に送るストリーミングと GraphQL データの埋め込みにより、表示開始が早まり、二重フェッチもなくなった。
- Watt の worker 監視により、1 つの重いレンダリングがプロセス全体を止めない。
- 大規模サービスでも Node + Fastify が本番の主力として選ばれた事例として注目される。

（元記事のコードは確認できなかったため、コード例は省略する。）

---

## 5. Web Worker API が Node の main ブランチでデフォルト有効化

**元記事:** [lib: enable web worker api by default · PR #65323](https://github.com/nodejs/node/pull/65323) / [worker: add support for Web Workers · PR #64894](https://github.com/nodejs/node/pull/64894)

### ① 目的・問題意識

ブラウザ・Deno・Bun ではグローバルの `Worker`（HTML 標準の Web Worker）が使えるが、Node では独自の `worker_threads` しかなかった。同じワーカーコードを複数の環境で動かすのが難しかった（長年の要望 [#43583](https://github.com/nodejs/node/issues/43583)）。

### ② 技術詳細

- **PR #64894**（Node 26.9.0 に入った）: HTML 標準の `Worker` を「Node が許す限り仕様に近く」実装した。当初は `--experimental-web-worker` フラグが必要だった。
- **PR #65323**（2026-11-11 に main へマージ、コミット `77e6379`、`semver-major`）: **フラグなしでデフォルト有効**にした。次のメジャー（Node 27 のアルファ）で使えるようになる見込み。初回使用時に実験的機能の警告は出ない。
- 実装の特徴（#64894 より）:
  - ワーカースクリプトはネットワークではなく**ローカルファイルシステムから同期的に読み込む**。
  - ワーカーのグローバルは通常の Node グローバルで、プロトタイプチェーンに `DedicatedWorkerGlobalScope` を追加している（仕様とは逆向きの構成）。
  - classic な `file:` ワーカーは CommonJS / ESM ローダーで実行し、`data:` / `blob:` ワーカーは classic script として実行する。
  - 内部は `worker_threads` を使っている。
- 制限:
  - `SharedWorker` は未実装。
  - `new Worker()` と `importScripts()` は `file:` / `data:` / `blob:` URL のみ受け付け、それ以外は `NotSupportedError`。
  - 読めないスクリプトは `NetworkError` となり、`error` イベントとして通知される。
  - オリジンモデルがないため、`file:` ワーカーの `location.origin` は `null`。
  - `close()` は仕様の closing flag アルゴリズムに従わず、即座に終了する。
  - `ErrorEvent` には `filename` / `lineno` / `colno` がない。
  - `languagechange` / `online` / `offline` / `rejectionhandled` / `unhandledrejection` イベントは発火しない。
  - メインスレッドでは相対 URL はカレントディレクトリ基準で、ワーカー内ではワーカー自身の URL 基準で解決される。
  - `WorkerOptions.credentials` は値の検証だけ行い、効果はない。
- 関連: [PR #66635](https://github.com/nodejs/node/pull/66635) で、グローバル `Worker` から起動したスレッド向けの `"worker"` パッケージ条件（exports condition）が提案されている。

使用例（PR には例がないため、標準 Web Worker API に沿ってこちらで書いた例）:

```js
// main.mjs — Node 27 以降（または 26.9+ で --experimental-web-worker）
const worker = new Worker(new URL('./worker.mjs', import.meta.url), { type: 'module' });
worker.onmessage = (e) => {
  console.log('result:', e.data);
  worker.terminate();
};
worker.onerror = (e) => console.error(e.message, e.error);
worker.postMessage(21);
```

```js
// worker.mjs — ブラウザと同じコードがそのまま動く
self.onmessage = (e) => {
  self.postMessage(e.data * 2);
};
```

### ③ なぜ効果的か

- **ブラウザ、Deno、Bun、Node でワーカーコードを共通化**できる。`require('worker_threads')` や `parentPort` を使う Node 専用の分岐が不要になる。
- 内部は既存の `worker_threads` なので、性能や安定性は既存と同等。

---

## 6. 組み込み YAML モジュールの提案（`node:yaml`）

**元記事:** [lib: add experimental node:yaml module · PR #67157](https://github.com/nodejs/node/pull/67157) / [Proposal: built-in YAML support, public API and placement · Issue #67347](https://github.com/nodejs/node/issues/67347)

### ① 目的・問題意識

YAML の設定ファイル（CI、Kubernetes、OpenAPI など）を読むだけでも、現状は `yaml` や `js-yaml` といった外部パーサーへの依存が必要になる。Node 自身の設定ファイルの議論（#53787）でも YAML が話題になった。

### ② 技術詳細

- フラグ: `--experimental-yaml`（`node:vfs` の導入方法 #63115 に倣う）。
- API: `parse(input[, options])`
  - 入力: 文字列、または UTF-8 のバイト列（`ArrayBuffer` / `SharedArrayBuffer` / TypedArray / `DataView`）。
  - 戻り値: ドキュメントが 1 つならその値、複数なら配列、空のストリームなら `null`。
  - 不正な YAML: `SyntaxError`（code `ERR_INVALID_YAML`、行と列つき）を投げる。`throwIfInvalid: false` なら `undefined` を返す（`fs.statSync()` の `throwIfNoEntry` に倣った設計）。

```js
const yaml = require('node:yaml');
yaml.parse('name: app\nports: [8080, 8443]\n');
// { name: 'app', ports: [ 8080, 8443 ] }
```

```js
import { parse } from 'node:yaml';

parse('port: 8080'); // { port: 8080 }
```

- **YAML 1.2 core schema** を採用:
  - `yes` / `no` は真偽値ではなく文字列（YAML 1.1 で有名な「ノルウェー問題」を回避）。
  - `0777` は 8 進数ではなく 10 進数。
  - タイムスタンプを Date に変換しない。マージキー（`<<`）を展開しない。`!!binary` をデコードしない。
  - アンカー、エイリアス、タグ（`!!str`、`!!int` など）、マルチドキュメントストリームに対応。
  - コレクションへのエイリアスは同じオブジェクトを指すので、結果に**循環参照**が含まれうる。
  - マッピングはプレーンオブジェクトになり、`__proto__` は自身のプロパティになる（プロトタイプ汚染の対策）。重複キーは `Duplicate mapping key` エラー。
  - `%YAML 1.1` ディレクティブは受け付けるが 1.2 のルールで読む。`%YAML 2.x` は拒否する。
- 実装:
  - 依存ゼロで一から書いた、テキストから値への**シングルパス**パーサー（`lib/yaml.js`、`lib/internal/yaml/parser.js`、約 1,500 行）。
  - マッピングはまず凍結した空プロトタイプの上で組み立て、完成後に `Object.prototype` を付け替える。これにより典型的なドキュメントで**約 60% 速くなった**と報告されている。
  - 公式 YAML Test Suite を JSON fixture として同梱し、402 件中 400 件が通る（失敗した 2JQS / X38W は重複キーのケース）。
- 未対応: `stringify`、`Blob` 入力、`.yaml` をモジュールとして import する機能。
- 今後の候補: `JSON.stringify` を手本にした `stringify(value[, replacer[, space]])`、`version` オプション、`reviver`、大きな入力向けのファイル / ストリーム API。
- 議論の論点:
  - モジュール名: `node:yaml` / `node:encoding/yaml` / `node:formats/yaml` で投票中。
  - 代替案: `util.parseYAML()`（`util.parseEnv()` と並べる）、`node:serdes`（`serdes.yaml.parse()`）、内部専用にとどめる、外部パッケージに任せる。
  - 「フォーマットごとにモジュールを作る前例になる」という懸念。panva からは、`node:logger` のように先に外部で安定させるべきではないかという意見も出ている。
  - PR は ShogunPanda、jasnell、cjihrig が承認済み（`semver-minor`、`notable-change`、`experimental`）。

### ③ なぜ効果的か

- 設定ファイルを読むだけのために依存を増やさずに済み、サプライチェーンのリスクも減る。
- 1.2 core schema、`__proto__` を自身のプロパティにする設計、重複キーのエラー化により、**安全で予測しやすいパース**になる。

---

## 7. Netlify Server が Early Access に

**元記事:** [Netlify Server is now available in Early Access (Changelog, 2026-11-10)](https://www.netlify.com/changelog/2026-11-10-netlify-server-early-access/) / [Netlify Server: long-running backends with WebSockets](https://www.netlify.com/blog/introducing-netlify-server/)（直接は読めず、検索結果から再構成した）

### ① 目的・問題意識

サーバーレス関数（短命なリクエスト / レスポンス）では、**WebSocket、ライブコラボレーション、長時間動く AI エージェント処理**のような常駐が必要なワークロードを扱いにくい。

### ② 技術詳細

- **長時間動く Node.js バックエンド**を Netlify 上で動かす。
- サーバは **`netlify/server/index.ts` の 1 ファイル**で定義する。プロビジョニング、スケーリング、監視は Netlify が行う。
- アイドル状態が **5 分**続くと **ゼロまでスケールダウン**する。Early Access 中は 1 サーバあたり最大 3 インスタンス（Free / Personal プランは 1）。
- 対応フレームワーク: **Express、Hono、Fastify**。Next.js、Astro、Nuxt はまだ Netlify Server 上では動かない。
- 料金: Early Access 中は無料（他の従量課金機能は通常どおり課金）。GA では予約メモリとアクティブ CPU 時間に課金する予定（料金未定）。クレジット制プランと、クレジットが有効な Enterprise プランで利用できる。API や挙動は今後変わる可能性がある。
- 最初の数日で 1 万件以上の Server プロジェクトが作られ、多くはエージェント経由だったと発表されている。

構成のイメージ（公式のサンプルコードは確認できなかったため、記事の説明に沿ってこちらで書いた例）:

```ts
// netlify/server/index.ts
import Fastify from 'fastify';
import websocket from '@fastify/websocket';

const app = Fastify();
await app.register(websocket);

app.get('/api/health', async () => ({ ok: true }));
app.get('/ws', { websocket: true }, (socket) => {
  socket.on('message', (msg) => socket.send(`echo: ${msg}`));
});

export default app; // エクスポート形式は公式ドキュメントで確認すること
```

### ③ なぜ効果的か

フロントエンドと同じリポジトリと同じデプロイの流れで、常駐型のバックエンド（WebSocket など）を持てる。ゼロまでスケールするので、アイドル時のコストも抑えられる。

---

## 8. Postgres.js 4.1

**元記事:** [porsager/postgres Releases](https://github.com/porsager/postgres/releases)

### ① 目的・問題意識

人気の Postgres クライアント `postgres`（Postgres.js）は、10/29 の 4.0 で公開 API 以下をすべて書き直した。続く 4.1 では運用上の課題（クエリのタイムアウト、まとめて送信、TLS 設定）を解決している。

### ② 技術詳細

#### 4.1.0（11/5）の新機能

- **`query_timeout`**（既定はオフ）: 長すぎるクエリをキャンセルし、`QUERY_TIMEOUT` で reject する。サーバやネットワークが無応答になった場合にも効く。
- **`cancel_timeout`**（既定 5 秒）: キャンセルしたクエリが終わるまで待つ上限。超えると接続を閉じる。
- **`sql.batch([queries])`**: 複数のクエリを **1 往復**で送る。トランザクション外では全体が一緒にコミットまたはロールバックされる。失敗したクエリの位置は `error.index` に入る。
- transform の **`json`** オプション: JSON 値を送信時にも変換するかどうか。
- **`eager_types`**: プール起動時に配列型とドメイン型を読み込む。
- **`sql.identifier(...parts)`**: 部品からクオート済みの識別子を作る。`sql.identifier('acme.inc', 'users')` → `"acme.inc"."users"`。
- **`.parsers(parsers)`**: そのクエリの行にだけ型パーサーを適用する。型 OID をキーにしたオブジェクト、またはカラムごとにパーサーを返す関数を渡す。`null` ならサーバのテキストのまま。
- **Node.js 18 を再サポート**（`engines: >=18`）。ただし `require` で読み込むには Node 20.19 / 22.12 以上が必要。

使用例（リリースノートの API 説明に沿ってこちらで書いた例）:

```js
import postgres from 'postgres';

const sql = postgres(process.env.DATABASE_URL, {
  query_timeout: 5_000,   // 5 秒を超えたらキャンセルして QUERY_TIMEOUT
  cancel_timeout: 5_000,  // キャンセルが返らなければ接続を破棄
});

// 1 往復でまとめて送る（トランザクション外なら全体で commit / rollback）
try {
  await sql.batch([
    sql`insert into logs ${sql({ msg: 'a' })}`,
    sql`update counters set n = n + 1 where id = ${id}`,
  ]);
} catch (err) {
  console.error('failed at query #', err.index);
}

// クオート済み識別子
await sql`select * from ${sql.identifier('acme.inc', 'users')}`;

// このクエリだけ numeric(OID 1700) を Number に変換する
const rows = await sql`select price from items`.parsers({ 1700: Number });
```

主な修正（4.1.0）:

- `PGSSLMODE` を読むようになった。**既存の接続に影響しうる**: `require` で TLS を使い、`verify-ca` / `verify-full` では Node の CA で検証する。
- URL 中の `sslrootcert` / `sslcert` / `sslkey` が、スタートアップパラメータとして送られず、正しく使われるようになった。
- `sql.listen` が最後の `unlisten()` の後も接続を保ち、失敗した再接続を再試行し続ける。
- サーバが終了させたトランザクションは、`CONNECTION_CLOSED` ではなくサーバの FATAL エラーで reject する。
- IP ホストの TLS 証明書を正しく検証する。キャッシュ済みのプリペアドステートメントがクエリ結果を保持し続ける問題を修正。

#### 4.1.1（11/11）の修正

- `COMMIT` / `ROLLBACK` の後に、await されなかった文が**トランザクション外で実行されてしまう**問題 → `UNSAFE_TRANSACTION` で reject するようにした。
- `sql.unsafe('select …; rollback')` で `ROLLBACK` の結果に `SELECT` のカラムが付く問題を修正。
- Cloudflare Hyperdrive で、型なしの文字列 / boolean パラメータを含むクエリの後ろに並んだクエリが解決しない問題を修正。
- `SHOW` / `EXPLAIN` などに対する `.cursor()` が行を返すようになった。
- `stack` を持たない接続エラー（Bun の `EAI_AGAIN` など）で、待機中のクエリがハングせず reject するようになった。
- キャンセルした `sql.batch` が `sql.end()` 後もプロセスを生かし続ける問題を修正。
- `_` で終わる JSON キーで `toCamel` / `toPascal` が例外を投げる問題を修正。

#### 参考: 4.0.0（10/29）の要点

- 破壊的変更:
  - パラメータの型は既定でサーバが決める（`infer: false`）。`Date` などはキャストが必要な場合がある（例: `${ date }::timestamptz`）。`infer: true` で v3 の挙動に戻る。
  - 対応ランタイムは Node 20+ / Bun / Cloudflare Workers。CJS ビルドは廃止され、`require('postgres').default` を使う。
  - `sql.array()` は廃止（`ARRAY_REMOVED`）。配列をそのまま渡し、`${ ids }::int[]` のようにキャストする。
- 新機能:
  - 最も早く終わる接続にクエリを割り当てるスケジューラ（`scheduler: false` で単純な方式に戻せる）。
  - `queue_timeout`、`.signal(abortSignal)`。
  - ネストした `sql.begin` はセーブポイントになる。
  - `sql.join`、名前付き引数、永続レプリケーションスロット。
  - SCRAM-SHA-256-PLUS（チャネルバインディング）、PostgreSQL 17+ の `sslnegotiation: 'direct'`。
  - `postgres:query` / `postgres:connect` / `postgres:transaction` の tracing channel。
- セキュリティ: 接続パラメータ中の NUL 文字によるスタートアップパラメータの注入を防止。`__proto__` カラムを通常の自身のプロパティにした。

### ③ なぜ効果的か

- `query_timeout` と `cancel_timeout` により、**無応答のサーバやネットワークでもハングしない**。
- `sql.batch` で往復回数が減り、レイテンシが下がる。巻頭記事の「ドライバの設定で最大 31%」という話にも通じる。
- 4.1.1 の `UNSAFE_TRANSACTION` は、await 忘れでトランザクション外にクエリが漏れる、気づきにくいバグを防ぐ。

---

## 参考リンク

- [Node Weekly #647](https://nodeweekly.com/issues/647)
- [PR #67288 green threads](https://github.com/nodejs/node/pull/67288) / [Green threads for Node.js](http://www.jasnell.me/posts/green-threads-in-node)
- [Node.js November 2026 Security Releases](https://nodejs.org/en/blog/vulnerability/november-2026-security-releases)
- [X: Rebuilding the global town square from scratch](https://blog.x.com/en/engineering/rebuilding-the-global-town-square-from-scratch)
- [PR #65323 enable web worker api by default](https://github.com/nodejs/node/pull/65323) / [PR #64894](https://github.com/nodejs/node/pull/64894)
- [PR #67157 node:yaml](https://github.com/nodejs/node/pull/67157) / [Issue #67347](https://github.com/nodejs/node/issues/67347)
- [Netlify Server Early Access](https://www.netlify.com/changelog/2026-11-10-netlify-server-early-access/)
- [Postgres.js releases](https://github.com/porsager/postgres/releases)
