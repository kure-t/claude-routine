# Node Weekly Issue #647 まとめ（2026年11月12日）

> 出典: [Issue #647: 22 Node, Bun and Deno stacks benchmarked — Node Weekly](https://nodeweekly.com/issues/647)
>
> **取得方法についての注意:** この環境からは nodeweekly.com・blog.x.com・netlify.com・macwright.com へ直接アクセスできなかった（DNS がブロックされている）。そのため、各記事の内容は WebSearch の結果と、直接取得できた GitHub ページ（nodejs/node の PR・Issue、porsager/postgres のリリースノート）から再構成した。
>
> - **「原文のコード」**と書いたコードは、PR や Issue の本文からそのまま写したもの。
> - **「説明用の例」**と書いたコードは、公開されている API 仕様をもとに本まとめで書いたもの。原文にはない。
>
> スポンサー記事は除外した。

---

## 目次

1. [22 通りの Node / Bun / Deno スタックのベンチマーク（巻頭記事）](#1-22-通りの-node--bun--deno-スタックのベンチマーク巻頭記事)
2. [Node.js のセキュリティリリース（2026年11月11日）](#2-nodejs-のセキュリティリリース2026年11月11日)
3. [Node に実験的な「グリーンスレッド」が登場（PR #67288）](#3-node-に実験的なグリーンスレッドが登場pr-67288)
4. [ブラウザ互換の Web Worker API が main ブランチでデフォルト有効に（PR #65323 / #64894）](#4-ブラウザ互換の-web-worker-api-が-main-ブランチでデフォルト有効にpr-65323--64894)
5. [X（旧 Twitter）が Web アプリを Fastify で作り直す](#5-x旧-twitterが-web-アプリを-fastify-で作り直す)
6. [組み込みの `node:yaml` モジュールの提案（PR #67157 / Issue #67347）](#6-組み込みの-nodeyaml-モジュールの提案pr-67157--issue-67347)
7. [Netlify Server がアーリーアクセスに：常駐型の Node バックエンド](#7-netlify-server-がアーリーアクセスに常駐型の-node-バックエンド)
8. [Val Town のサービス終了と、Tom MacWright「Deno のセキュリティモデルが好きだった」](#8-val-town-のサービス終了とtom-macwrightdeno-のセキュリティモデルが好きだった)
9. [Postgres.js 4.1：`sql.batch()` と `query_timeout`](#9-postgresjs-41sqlbatch-と-query_timeout)

---

## 1. 22 通りの Node / Bun / Deno スタックのベンチマーク（巻頭記事）

**リンク先:** YouTube 動画「I tested EVERY Javascript runtime (surprising)」（https://www.youtube.com/watch?v=zDw_DxM5GlI）

> リンク先が動画のため、指示どおり動画の中身は確認していない。以下は Node Weekly の紹介文と検索結果から分かる範囲の内容。

### ① 目的・問題意識

「どのランタイムを使うか」と「どの Web フレームワークを使うか」の比較記事はたくさんある。ただ、その多くは "Hello World" を返すだけのマイクロベンチマークで、DB アクセスを含む実際のアプリの性能とはずれがちだった。この検証は、**現実的な API アプリを同じ仕様で 22 通りに実装し**、ランタイムとフレームワークのどちらが性能を大きく左右するのかを確かめている。

### ② 技術詳細

- **題材:** Medium のクローン。エンドポイントは 19 個で、Postgres にはシードデータが入っている。
- **組み合わせ:** Node・Bun・Deno のそれぞれで、人気のあるフレームワークを使って同じアプリを作った（合計 22 スタック）。
- **公開物:** コードと全結果は GitHub で公開されている（リポジトリ名は検索では確認できなかった）。
- **結論:** **フレームワークよりもランタイムの選択の方が性能差に効く。**

参考として、同じ時期に GitHub で公開された別のベンチマークも検索で見つかった。こちらは 22 スタックの検証とは別のもの。

- `mochavin/js-runtime-bench`: 同じ API を同じ k6 負荷試験で比較したもの。結果の日付は 2026-11-07。シングルプロセスのスループットは Bun が約 6,288 req/s、Deno が約 6,086 req/s、Node が約 5,337 req/s。ワーカーを増やしても全体のスループットはほとんど伸びなかったが、計算の重い処理のレイテンシはおおよそ半分になった。
- `aurora-algorithm/js-runtimes-bench`: `node:http`・`Bun.serve`・`Deno.serve`・Express・Postgres を比較したもの。

### ③ なぜ重要か

- 本物の DB と多数のエンドポイントを含めると、フレームワーク固有のオーバーヘッドは処理全体の中で目立たなくなる。そのかわりに、ランタイムの I/O 実装・HTTP パーサー・DB ドライバーの差が表に出てくる。
- 実務では「速いフレームワークへ乗り換える」より、「ランタイムと DB ドライバーを見直す」方が効果が大きいことがある。その判断材料になる。

---

## 2. Node.js のセキュリティリリース（2026年11月11日）

**関連リンク:** Node.js 公式のセキュリティリリース告知（nodejs.org/en/blog）。解説記事として [The Hacker News](https://thehackernews.com/2026/11/nodejs-fixes-16-flaws-including-http2.html) など。

### ① 目的・問題意識

HTTP/2 と TLS/HTTPS のサーバー実装を中心に、**16 件の脆弱性**が修正された。深刻度の内訳は High が 7 件、Medium が 6 件、Low が 3 件。認証なしのクライアントからサーバーを落とせるものや、他の接続のメモリを読めるものが含まれている。

### ② 技術詳細

**修正版:** `v22.23.5` / `v24.22.3` / `v26.13.1`。Node 20 はすでに EOL なので修正は出ず、「EOL 版は常に影響を受ける」扱いになる。

| CVE | 種類 | 内容 |
|---|---|---|
| CVE-2026-123864 | HTTPS の再入（re-entrancy）バグ（High） | 認証なしのクライアントが**他の接続のメモリ**を読める。読めるものには TLS 秘密鍵や他ユーザーのリクエストが含まれる |
| CVE-2026-123863 | HTTP/2 セッション終了時に到達できる assertion（High） | デフォルト設定の HTTP/2 サーバーで、リクエストボディを受け取り終わる前に応答すると、クライアントがプロセスを abort させられる。TLS・平文のどちらでも起きる |
| CVE-2026-123865 | HTTP/2 のシャットダウン時のリソースリーク | 終了処理でリソースが解放されない |
| CVE-2026-126051 | TLS サーバーのハンドシェイク中の未捕捉例外（High） | 24.x と 26.x が対象。遅延コールバックや `server.addContext()` を使うサービスでプロセスがクラッシュする |
| CVE-2026-123873 | Permission Model（Low） | ヒープスナップショットの出力先のチェック |
| CVE-2026-106071 | Permission Model（Low） | ワイルドカード指定の権限の扱い（解説記事によっては記載がない） |

※ CVE ごとの記述は解説記事によって細かく食い違う。正確な内容は公式アドバイザリで確認すること。

**バージョン確認の例（説明用の例）:**

```bash
node -v
# v22.23.5 / v24.22.3 / v26.13.1 以上になっているかを確認する
```

```js
// 起動時に修正版未満なら警告を出す（説明用の例）
const [major, minor, patch] = process.versions.node.split('.').map(Number);
const fixed = { 22: [23, 5], 24: [22, 3], 26: [13, 1] };
const f = fixed[major];
if (!f || minor < f[0] || (minor === f[0] && patch < f[1])) {
  console.warn(`Node ${process.version} は 2026-11 のセキュリティ修正を含みません`);
}
```

### ③ なぜ重要か

- HTTP/2・HTTPS サーバーは、インターネットに直接公開されることが多い部分。認証なしで DoS や秘密鍵の漏洩が起きうるので、**すぐにアップデートすべき**内容になっている。
- `--permission` を使っている場合は、アドバイザリとリリースノートの両方を確認するよう勧められている。リリースノートには、アドバイザリに書かれていない Permission Model の修正が含まれているため。

---

## 3. Node に実験的な「グリーンスレッド」が登場（PR #67288）

**リンク先:** [src: experimental green threads implementation · nodejs/node#67288](https://github.com/nodejs/node/pull/67288)

PR の状態: Open。2026-11-14 にレビュー可能になった。mcollina と ronag が承認済み。ラベルは `experimental` と `large-pr`。

### ① 目的・問題意識

プラグイン、テナントごとのハンドラー、ユーザースクリプトのように、**小さくて信頼できないコードを大量に実行したい**場面がある。既存の手段にはそれぞれ問題があった。

- **`worker_threads`:** ワーカー 1 つにつき OS スレッドとイベントループを 1 つずつ持つ。起動が重く、メモリも食う（1 インスタンスあたり約 6 MB）。1,000 テナントなら OS スレッドが 1,000 本になる。
- **`vm` コンテキスト:** 軽いが、**セキュリティ境界ではない**。同期的な無限ループやヒープの使い切りが、同じワーカー上の他テナントにまで波及する。

グリーンスレッドはこの中間を狙っている。**テナントごとに独立した V8 Isolate を持たせつつ、少数の OS スレッドを共有する**、軽量なワーカーである。

### ② 技術詳細

**設計の要点:**

- **Isolate:** 各グリーンスレッドは独自の V8 Isolate を持つ。使えるのは ECMAScript の組み込み機能と、最小限のグローバル（`postMessage`、`onmessage`、`close()`、`queueMicrotask()`、タイマー、ホストへ転送される `console`）だけ。**Node の API（`fs`・`net`・`child_process` など）は使えない。**
- **スレッドプール:** プロセス全体で 1 つのプールを共有し、各スレッドが libuv ループを持つ。デフォルトのサイズは `min(8, availableParallelism())` で、環境変数 `NODE_GREEN_THREADS_POOL_SIZE` で変えられる。
- **スケジューリング:** Isolate ごとに FIFO の作業キューがある。その Isolate のロックを持つプールスレッドが一区切り分の処理を実行し、続けてマイクロタスクのチェックポイントを処理する。どのスレッドにも紐付かない作業はプロセス全体の ready キューに入り、I/O に紐付いた作業はその I/O を持つスレッドに残る。
- **呼び出しコンテキスト:** メッセージや評価は、それぞれ専用のコンテキストで実行される。タイマーや Promise の reaction には、continuation-preserved embedder data でそのコンテキストが引き継がれる。
- **リソース制限:** プロセス全体で 1 本の watchdog スレッドが CPU 予算を監視し、`maxCpuTimeMs`（累計）か `maxTaskCpuTimeMs`（タスク単位）を超えたら割り込みをかける。`maxMemoryMb` はヒープと ArrayBuffer を合わせた上限。
- **割り込みが効かない Isolate への対処:** V8 が割り込みを無視するコードを JIT コンパイルしてしまった場合は、予備の watchdog が停止要求から 1 秒待つ。それでも止まらなければ、そのグリーンスレッドを放棄して `GreenThreadWarning` を出し、終了コード 1 で終わらせる。具体的には、V8 が生成したコードの実行中にシグナルが届くまでプールスレッドにシグナルを送り続け、届いたらプールのループへジャンプして戻る。これが失敗するか 10 秒を超えたら、プロセスを abort する。
- **Isolate グループ:** ポインタ圧縮が有効な環境では、デフォルトでグリーンスレッドごとに別の Isolate グループを使う。`isolateGroup: 'default'` や、共有の `IsolateGroup` も指定できる。
- **スナップショットとウォームプール:** Node 本体より小さい起動スナップショットを使う。`setWarmPoolSize(n)` で Isolate を事前に作っておける。戻り値は、プールが埋まったときに resolve する Promise。
- **その他:** `BroadcastChannel` は `broadcastScopes` に列挙した名前だけで使える。`close()` で正常終了できる。ホスト側から返るエラーは、データプロパティだけから組み立て直される。
- **フラグ:** `--experimental-green-threads` を付けたときだけ有効。

**基本的な使い方（原文のコード）:**

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

**プラグインホストの例（原文のコード）:**

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

このコードのポイント:

- `env` に渡した関数だけが、プラグインから呼べる「能力（capability）」になる。プラグインは、ホストが渡したもの以外に触れない。
- `plugin.exports` で、プラグインがエクスポートした関数を **RPC スタブ**として受け取り、`await transform(...)` のように呼び出す。
- `resourceLimits` で、1 タスクあたり CPU 100ms、メモリ 32MB に制限している。

**ベンチマーク（88 CPU の Linux x64 での作者の計測。単発の実行なので細かい差はノイズ）:**

| 指標（1,000 インスタンス） | Green thread | Worker |
|---|---|---|
| 起動数/秒（コールド） | 約 3,739 | 約 977 |
| 1 インスタンスあたりのメモリ（PSS） | 1.36 MB | 6.17 MB |
| OS スレッド数 | 8（共有） | 1,000 |
| ウォームプールからの起動数/秒 | 約 80,778 | 該当なし |

- 1,000 テナントに 5 万リクエストを送った場合: グリーンスレッドは約 202k req/s、Worker は約 157k req/s。
- メッセージの往復時間はほぼ同じ（約 20µs）。並列度をそろえた CPU バウンド処理（`fib(27)`）もほぼ同じ。
- 100 テナントのうち 1 つが無限ループに入った場合: グリーンスレッドは約 52ms で停止させ、その間に他のテナントは 200ms で約 46k リクエストをさばいた。Worker は停止に約 65ms かかり、他のテナントがさばけたのは約 36k リクエスト。

**`vm` コンテキストを Worker プールで動かす方式との比較（レビュアーの devsnek が提案した方式）:**

- テナントあたりのコストは `vm` の方が軽い（0.25MB 対 1.36MB）。コールドスタートも速い（8 ワーカーで約 15k/s 対 約 3.8k/s）。
- タイムアウトなしの素のスループットはほぼ同じ（約 190k req/s）。ただし `vm` 側で `timeout` を付けると約 80k req/s まで落ちる。
- `vm` では、同期的な無限ループが同じワーカーの他テナントを遅らせる（他テナントの最も遅いリクエストが 52ms。グリーンスレッドでは 3ms）。
- `vm` では、マイクロタスクのループが `timeout` をすり抜ける（`microtaskMode: 'afterEvaluate'` を指定すれば防げる）。止めるにはワーカーごと kill するしかなく、巻き添えで 13 テナントが落ちる。
- `vm` では、1 テナントがヒープを使い切るとワーカー全体が落ちる。

**制限事項:**

- スレッドセーフな共有オブジェクトはない。値は structured clone でコピーされ、オブジェクトは RPC スタブ経由でやり取りする。
- 並列に動けるのはプールサイズ（デフォルト 8）まで。
- `Atomics.wait()` のような同期待ちは無効。
- V8 のバグにより、JIT コンパイルされた一部のループは終了要求を無視する。放棄された Isolate のメモリは解放されない。
- タイミング攻撃などのサイドチャネルには対策していない。`SharedArrayBuffer` を渡すと、高精度タイマーとして使えてしまう。
- 今後追加予定: `TextEncoder` / `TextDecoder`、`URL` / `URLSearchParams`、`structuredClone()`。

**レビューで出た論点:**

- `--permission` 下で Worker は `--allow-worker` を要求するのに、`GreenThread` には権限チェックがない。同じ権限で制御すべきでは、という指摘。
- `SharedArrayBuffer` のフラグをスピン待ちすると、プール内で OS がプリエンプトできず、フラグを立てるはずの側が永遠に動けなくなる（デッドロック）。再現用の gist が添えられている。
- 送信に失敗した場合（引数に Symbol が含まれていたときなど）、登録済みのエクスポート関数が解放されないという指摘。
- いつ Worker ではなくグリーンスレッドを使うべきかのドキュメントがほしい、という要望。

### ③ なぜ効果的か

- **M:N スケジューリング:** 多数の Isolate（M）を少数の OS スレッド（N）に載せる。スレッド生成とイベントループの初期化コストがなくなり、起動数は約 4 倍、メモリは約 1/4.5 になった。
- **Isolate 単位で隔離される:** ヒープが別なので、`vm` と違って 1 テナントの OOM や無限ループが他テナントに波及しない。
- **capability ベースのセキュリティ:** Node の API を最初から持たせず、ホストが `env` で明示的に渡したものだけを使わせる。「何ができるか」をホストが完全に決められる。
- **ウォームプール:** Isolate を事前に作っておくことで、起動数は約 8 万/秒に達する。リクエストごとに使い捨ての実行環境を作る設計が現実的になる。

---

## 4. ブラウザ互換の Web Worker API が main ブランチでデフォルト有効に（PR #65323 / #64894）

**リンク先:**

- [lib: enable web worker api by default · nodejs/node#65323](https://github.com/nodejs/node/pull/65323)（2026-11-11 にマージ。コミット `77e6379`。`semver-major`）
- [worker: add support for Web Workers · nodejs/node#64894](https://github.com/nodejs/node/pull/64894)（Node 26.9.0 で実験的に入った）

### ① 目的・問題意識

Node の `worker_threads` は Node 独自の API で、ブラウザの `new Worker(url)` / `postMessage` / `onmessage` とは形が違う。そのため、同じワーカーのコードをブラウザ・Deno・Bun と Node で共有しにくかった（Issue #43583）。PR #64894 で **HTML 標準に沿った Web Worker API** が `--experimental-web-worker` フラグ付きで入り、今回の PR #65323 で**フラグなしでも使えるデフォルト有効**になった。Node 27 の今後のアルファ版に入る見込み。

### ② 技術詳細

**変更前のドキュメント（原文のコード）:**

```
## Class: `Worker`

<!-- YAML

added: REPLACEME

-->

> Stability: 1 - Experimental. Enable this API with the
> [`--experimental-web-worker`][] CLI flag.
```

**ブラウザと同じ書き方で使う例（説明用の例）:**

```js
// main.mjs  — Node 27 系（main ブランチ）ならフラグ不要
// Node 26.9+ では `node --experimental-web-worker main.mjs`
const worker = new Worker(new URL('./worker.mjs', import.meta.url), { type: 'module' });

worker.onmessage = (event) => {
  console.log('result:', event.data); // result: 42
  worker.terminate();
};
worker.onerror = (event) => console.error(event.message, event.error);

worker.postMessage(21);
```

```js
// worker.mjs
self.onmessage = (event) => {
  self.postMessage(event.data * 2);
};
```

**仕様（HTML 標準）との違い（PR #64894 より）:**

- **`SharedWorker` はない。** 共有の仕組みがオリジンとブラウジングコンテキストに依存しており、Node にはその概念がないため。
- **スクリプトは同期的にローカルから読み込む。**
  - `new Worker()` と `importScripts()` が受け付けるのは `file:`・`data:`・`blob:` の URL だけ。それ以外のスキームは `NotSupportedError` を投げる。
  - スクリプトが読めない場合は `NetworkError` を投げる。仕様では `error` イベントになるところ。
  - リダイレクト、`nosniff`、HTTP の MIME 検証は適用されない。MIME 検証は `data:` と `blob:` だけ。
  - `WorkerOptions.credentials` は検証されるだけで、効果はない。
- **オリジンモデルがない。** `file:` ワーカーでは `location.origin` が `null` になる。
- **`close()` は即座に終了する。** 仕様のような closing フラグのアルゴリズムは使わないので、`close()` の後に書いた同じタスク内のコードは実行されない。
- **グローバルオブジェクトの構造が逆。** Node の通常のグローバルのプロトタイプチェーンに `DedicatedWorkerGlobalScope` を足している（仕様では逆の関係）。クラシックな `file:` ワーカーは CJS / ESM のローダー経由で実行されるので、トップレベルの宣言はグローバルにならない。`data:` / `blob:` のクラシックワーカーは、従来どおりクラシックスクリプトとして実行される。
- **`ErrorEvent` は一部だけ。** `message` と `error` はあるが、`filename`・`lineno`・`colno` はない。ワーカー内の未処理エラーはそれ以上伝播しない。どちらも、内部で使っている `worker_threads` 実装の制約による。
- **発火しないイベント:** `languagechange`・`online`・`offline`（概念がない）と、`rejectionhandled`・`unhandledrejection`（`PromiseRejectionEvent` がない）。
- **相対 URL の解決基準:** メインスレッドではカレントディレクトリを基準に解決する（ドキュメントのベース URL がないため）。ワーカー内では仕様どおり、ワーカー自身の URL が基準。

**関連 PR:** #66635（guybedford）で、Web Worker スレッド向けの `"worker"` 条件付きエクスポートを追加する提案が出ている。

**レビューでのやり取り:** 初回使用時に実験的機能の警告が出るかという質問（mcollina）に対し、現状は出ず、追加するなら別の PR で、という回答（panva）があった。

**テスト:** WPT（Web Platform Tests）で検証されている。パッチのカバレッジは 93.64%。

### ③ なぜ効果的か

- **同じワーカーのコードがどこでも動く:** ブラウザ・Deno・Bun・Node で同じ `new Worker(url)` のコードが動き、`worker_threads` 用の分岐やポリフィルがいらなくなる。
- **WPT で互換性を担保している:** ブラウザと同じテストスイートで検証しているので、挙動の差が少ない。差がある部分も、上のように明文化されている。
- **ネットワーク読み込みをしない:** `file:`・`data:`・`blob:` だけに制限することで、サーバー環境で意図しないリモートコードを読み込むのを防いでいる。

---

## 5. X（旧 Twitter）が Web アプリを Fastify で作り直す

**リンク先:** [Rebuilding the global town square from scratch — X Engineering Blog](https://blog.x.com/en/engineering/rebuilding-the-global-town-square-from-scratch)（直接は取得できず、検索結果と二次記事から再構成）

### ① 目的・問題意識

以前の X の Web アプリは、**一般ユーザー向け・検索クローラー向け・リンクプレビュー向けに 3 種類のサーバー**を運用していた。さらに、回線が遅いと、プロフィールの最初のポストが表示されるまで **22 秒以上**かかっていた（Slow 4G で計測）。

### ② 技術詳細

- **スタック:** Vite、React、**Fastify**、**Platformatic Watt**、Relay と GraphQL、XDS（X の新しいコンポーネントライブラリ）。これらを X 独自のサーバーパッケージでまとめている。
- **Fastify の役割:** ミドルウェアから最終的な HTML ストリームまで、**すべてのリクエストを Fastify が処理する**。
- **Platformatic Watt の役割:** Node.js v26 上で、サーバーを **1 プロセス内の複数のワーカースレッド**として動かす。イベントループの遅延やメモリが上限を超え続けたワーカーは入れ替えられる。そのため、1 つのレンダリングが固まっても、影響はそのワーカーが受け持つリクエストだけで済む。
- **ストリーミング SSR:** React のストリーミングサーバーレンダリングと Suspense を使う。サーバーはページのシェルができた時点ですぐ送り、各セクションはデータが届いた順にストリームで追加していく。
- **GraphQL レスポンスの再利用:** サーバーが受け取った GraphQL レスポンスはページに一度だけ埋め込まれる。ブラウザはそれを再利用するので、同じ API を問い合わせ直さない（Relay のストアに注入する）。
- **クローラー対応:** 検証済みの検索クローラーには、クライアントバンドルや不要なルーター状態を含まない、完全にレンダリングされた HTML を返す。サイズは Google の 2MB 制限に収まるように調整している。
- **統合:** 3 種類あったサーバーを 1 つにまとめた。
- **成果:** 遅い回線でも、最初のポストの表示が **2.7 秒**になった（以前は 22 秒以上。23.9 秒とする記事もある）。数字は X 自身の計測（2026-11-03、Slow 4G、3 回実行した中央値）。
- **体制:** エンジニア 5 人が AI コーディングエージェントと一緒に作業し、最初のバージョンを 6 か月以内に作った。
- **展開状況:** ログアウト状態のページとクローラー向けのトラフィックはすでに新アプリで動いている。ログインユーザーは国ごとに順次移行している。

**構成のイメージ（説明用の例。X の実際のコードではない）:**

```js
// server.mjs — Fastify + React のストリーミング SSR の基本形
import Fastify from 'fastify';
import { renderToPipeableStream } from 'react-dom/server';
import { PassThrough } from 'node:stream';
import App from './App.js';

const app = Fastify({ logger: true });

app.get('/*', (request, reply) => {
  const relayPayloads = []; // サーバーで受け取った GraphQL レスポンスを貯める
  const stream = new PassThrough();

  const { pipe } = renderToPipeableStream(<App url={request.url} relayPayloads={relayPayloads} />, {
    bootstrapScripts: ['/assets/client.js'],
    onShellReady() {
      // シェルができたらすぐ送り始める（TTFB を短くする）
      reply.type('text/html').send(stream);
      pipe(stream);
    },
    onShellError(err) {
      reply.code(500).send('Internal Error');
    },
  });
});

await app.listen({ port: 3000 });
```

```json
// watt.json — Platformatic Watt で同じアプリを複数のワーカースレッドで動かす例（説明用の例）
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/runtime/2.0.0.json",
  "entrypoint": "web",
  "workers": 4,
  "health": { "maxELU": 0.95, "maxHeapUsed": 0.9 },
  "services": [{ "id": "web", "path": "./web" }]
}
```

### ③ なぜ効果的か

- **ストリーミングと Suspense:** データ取得を待たずにシェルを返すので、体感速度（FCP / LCP）が大きく上がる。
- **GraphQL レスポンスを 1 回だけ埋め込む:** ハイドレーション時に同じデータを二重に取りに行かずに済み、帯域と待ち時間を節約できる。
- **Watt で 1 プロセス・複数スレッド:** マルチプロセスより軽い。固まったワーカーは自動で入れ替わるので、障害の影響をワーカー単位に閉じ込められる。
- **3 サーバーを 1 つに統合:** レンダリングのロジックが 1 か所にまとまり、ユーザー・クローラー・プレビューの間で表示が食い違わなくなる。運用コストも下がる。

---

## 6. 組み込みの `node:yaml` モジュールの提案（PR #67157 / Issue #67347）

**リンク先:**

- [lib: add experimental node:yaml module · nodejs/node#67157](https://github.com/nodejs/node/pull/67157)（Open。ShogunPanda、jasnell、cjihrig が承認）
- [Proposal: built-in YAML support, public API and placement · nodejs/node#67347](https://github.com/nodejs/node/issues/67347)

### ① 目的・問題意識

Node 自身の設定ファイル（`.noderc` / `--experimental-config-file`）の議論（Issue #53787）では、**コアに YAML パーサーがない**ことが理由で、形式が JSON のみになった。一方、Bun は v1.2.21 から `Bun.YAML.parse()` を提供している。YAML は設定ファイルの事実上の標準なので、コアで読めるようにしたい、というのが提案の動機。

### ② 技術詳細

**基本的な使い方（原文のコード）:**

```js
const yaml = require('node:yaml');
yaml.parse('name: app\nports: [8080, 8443]\n');
// { name: 'app', ports: [ 8080, 8443 ] }
```

```js
import { parse } from 'node:yaml';

parse('port: 8080'); // { port: 8080 }
```

**API の仕様: `parse(input[, options])`**

- **入力:** 文字列、または UTF-8 のバイト列（`ArrayBuffer`・`SharedArrayBuffer`・TypedArray・`DataView`）。
- **戻り値:** 単一のドキュメントならその値、複数のドキュメントからなるストリームなら配列、空のストリームなら `null`。
- **エラー:** `SyntaxError` を投げる。`code` は `ERR_INVALID_YAML` で、行と列の情報が付く。`throwIfInvalid: false` を指定すると、投げずに `undefined` を返す（`fs.statSync()` の `throwIfNoEntry` にならったもの）。
- **スキーマ:** YAML 1.2 の core schema。`yes` / `no` は文字列のままで、`0777` は 10 進数として読む。
- **未対応のもの:** タイムスタンプから Date への変換、マージキー（`<<`）の展開、`!!binary` のデコード。
- **アンカーとエイリアス:** エイリアスは同じオブジェクトを指すので、結果に循環参照が含まれることがある。
- **キーの重複:** `Duplicate mapping key` エラーになる（`yaml` や `js-yaml` パッケージのデフォルトと同じ）。
- **`__proto__` キー:** 自身のプロパティとして保持される（プロトタイプ汚染を防ぐ）。
- **バージョン指定:** `%YAML 1.1` ディレクティブは受け付けるが、1.2 のルールで読む。`%YAML 2.x` は拒否する。
- **フラグ:** `--experimental-yaml`（`node:vfs` の PR #63115 と同じ仕組み）。

**エラー処理の例（説明用の例）:**

```js
// node --experimental-yaml config.mjs
import { parse } from 'node:yaml';
import { readFileSync } from 'node:fs';

const buf = readFileSync('./config.yaml');      // Buffer をそのまま渡せる
try {
  const config = parse(buf);
  console.log(config);
} catch (err) {
  if (err.code === 'ERR_INVALID_YAML') {
    console.error(`YAML 構文エラー: ${err.message}`); // 行と列の情報が含まれる
  }
}

// 例外を投げないモード: エラーオブジェクトを作らないぶん速い
const maybe = parse('a: [1, 2', { throwIfInvalid: false });
console.log(maybe); // undefined
```

**実装上の工夫:**

- `lib/internal/yaml/parser.js` に、**外部依存なしで JS で書いた 1 パスのパーサー**を置いている。最初のエラーで内部の番兵（sentinel）を使って止まるので、例外を投げないモードではエラーメッセージ自体を組み立てない。
- **マッピングの組み立て方:** 凍結した空のプロトタイプでオブジェクトを作り、エントリを代入し終えてから最後に一度だけ `Object.prototype` を設定する。プロパティを 1 つずつ定義する方式より、一般的なドキュメントで**約 60% 速い**。結果のオブジェクトは V8 の fast properties を保つ。
- バッファの入力は、使い回しの `TextDecoder` 1 つでデコードする。

**テスト:** 公式の YAML Test Suite を、リビジョンを固定した JSON fixture として取り込んでいる。402 ケース中 400 ケースが通る。失敗する 2JQS と X38W はキーが重複したドキュメントで、テストスイート側に期待される出力がない。fixture は `tools/update-yaml-test-suite.mjs` で再生成できる。

**ベンチマーク（1 回のパースにかかるミリ秒。JS の結果オブジェクトを作るところまで含む）:**

| パーサー | 1 KiB | 17 KiB | 332 KiB |
|---|---|---|---|
| `node:yaml`（この PR） | 0.016 | 0.36 | 6.49 |
| rapidyaml（C++ アドオン） | 0.011 | 0.16 | 3.54 |
| yaml-rust2（Wasm） | 0.024 | 0.40 | 7.81 |
| `JSON.parse` | 0.003 | 0.063 | 1.26 |

C++ にしても 17KiB あたり約 0.2ms しか縮まず、新しいネイティブ依存を抱える理由にならないとして、JS 実装を採ることになった。`js-yaml` はほぼ同じ速さだが、デフォルトで YAML 1.1 の型解決をする。`yaml` パッケージはこの用途には大きすぎ、約 15 倍遅い。

**レビューでの論点:**

- **モジュール名:** `node:yaml`、`node:formats/yaml`（jasnell が提案し、ShogunPanda が支持）、`node:parser`、`node:encoding/yaml`、`node:markup`、`node:serialization` などが挙がった。
- **バージョン:** 将来の YAML 1.3 に備えて `version` オプションを予約しておくべきか。
- **大きなファイル:** メモリの急増を避けるため、ストリーミングで読む `parseFile()` がほしいという意見。
- **YAML より TOML を:** TOML の方が単純だという意見に対し、YAML の方が広く使われているという反論。
- **いきなりコアに入れるべきか:** `node:logger` のように、まず userland のパッケージで成熟させるべきではないか、という質問（panva）。

**この提案が通った場合の次の段階:**

1. Bun と同じ仕様の `yaml.stringify(value, replacer, space)`
2. `import data from './config.yaml' with { type: 'yaml' }`
3. `--experimental-config-file` での YAML 対応
4. 実験的フラグの解除

### ③ なぜ効果的か

- **依存が減る:** 設定ファイルを読むためだけに `yaml` や `js-yaml` を入れる必要がなくなる。サプライチェーンの面でもインストールサイズの面でも有利。
- **安全なデフォルト:** 1.2 の core schema を使い、`__proto__` を自身のプロパティとして扱い、キーの重複をエラーにする。`no` が false になるような、YAML 1.1 由来の思わぬ型変換やプロトタイプ汚染を防げる。
- **Bun との互換:** API の形を `Bun.YAML.parse()` にそろえているので、ランタイムをまたいでコードを共有しやすい。

---

## 7. Netlify Server がアーリーアクセスに：常駐型の Node バックエンド

**リンク先:**

- [Netlify Server: long-running backends with WebSockets](https://www.netlify.com/blog/introducing-netlify-server/)
- [Changelog（2026-11-10）](https://www.netlify.com/changelog/2026-11-10-netlify-server-early-access/)
- [Docs](https://docs.netlify.com/build/server/)

（直接は取得できず、検索結果から再構成）

### ① 目的・問題意識

Netlify のバックエンドは、これまで Functions（サーバーレス）が中心だった。そのため、**WebSocket や SSE のような長時間の接続、リクエストをまたいだメモリ上の状態、任意の Node フレームワークをそのまま動かすこと**が難しかった。Netlify Server は、**長時間動き続ける Node.js サーバー**を Netlify 上で動かせるようにする。

### ② 技術詳細

- **常駐する Node.js サーバー:** WebSocket と SSE の接続を保持でき、リクエストをまたいで状態を持てる。1 インスタンスで多数のリクエストを同時に処理し、自動でスケールする。
- **アダプター不要:** どの Node.js の Web フレームワーク・サーバーも、**Netlify 専用のアダプターやラッパーなしで**そのまま動く。
- **スケーリング:** 最大 3 インスタンスまで（Free / Personal プランは 1）。リクエストが 5 分間ないと必ずゼロまでスケールダウンする。
- **提供条件:** クレジット制のプランと、クレジットを有効にした Enterprise プランで使える。ローカルでビルドするには Netlify CLI 27.17.0 以上が必要。
- **料金:** アーリーアクセス中は無料。GA 後は、予約したメモリとアクティブな CPU 時間に応じてクレジットを消費する。
- **未対応:** Next.js・Astro・Nuxt のようなフルスタックフレームワークはまだ動かない。API や挙動はアーリーアクセス中に変わる可能性がある。

**「そのまま動く」サーバーの例（説明用の例。普通の Node サーバーで、Netlify 固有のコードはない）:**

```js
// server.mjs
import Fastify from 'fastify';
import websocket from '@fastify/websocket';

const app = Fastify();
await app.register(websocket);

const clients = new Set();          // リクエストをまたいで保持する状態
app.get('/ws', { websocket: true }, (socket) => {
  clients.add(socket);
  socket.on('message', (msg) => {
    for (const c of clients) c.send(msg.toString()); // ブロードキャスト
  });
  socket.on('close', () => clients.delete(socket));
});

app.get('/health', async () => ({ ok: true, clients: clients.size }));

await app.listen({ port: Number(process.env.PORT ?? 3000), host: '0.0.0.0' });
```

### ③ なぜ効果的か

- サーバーレスでは難しいチャット、通知、リアルタイムダッシュボードを、フロントエンドと同じ Netlify のプロジェクトで運用できる。
- アダプターがいらないので、ベンダーロックインを避けつつ、既存の Express / Fastify / Hono のアプリを移せる。
- アイドル 5 分でゼロまで縮むので、トラフィックの少ないサービスでもコストを抑えられる。

---

## 8. Val Town のサービス終了と、Tom MacWright「Deno のセキュリティモデルが好きだった」

**リンク先:**

- [I was a fan of Deno's security model — macwright.com（2026-11-09）](https://macwright.com/2026/11/09/deno-security-model)
- 背景: [Val Town 2023-2025 Retrospective](https://macwright.com/2025/11/11/val-town) / [The first four Val Town runtimes](https://blog.val.town/first-four-val-town-runtimes)

### ① 目的・問題意識

**Val Town のサービス終了:** 2026-11-10、創業者の Steve Krouse がホスティングサービスの終了を発表した。理由はクラウド市場で競争できなかったため。発表と同時に新規登録とプランのアップグレードは止まった。

- 2026-12-10: 読み取り専用になる（SQLite と Blob ストレージは対象外）
- 2027-01-10: その他のサービスもすべて終了（デプロイ済みのアプリはこの日まで動く）
- 2027-02-01: Val Town をオープンソース化
- 移行先: 公式の移行ガイドは、機能が最も近い Cloudflare を勧めている。Vercel、Netlify、Render も候補に挙がっている。

**MacWright の記事:** 「Deno のパーミッションモデルは役に立たない」という批判記事への反論として書かれた。他人のコードを実行するサービスでは、サンドボックスの強さが事業の生命線になる。

### ② 技術詳細

- **vm2 は罠だった:** MacWright が参加した当初、Val Town はユーザーの JavaScript を `vm2` で実行していた。vm2 にはサンドボックスを抜け出す方法がいくつもあり、他人のコードやデータに届いてしまう。その後、メンテナー自身が vm2 を非推奨にした。
- **Deno への移行:** Deno のランタイムフラグによるパーミッションに、**会社の命運を賭けた**。その結果、Deno のフラグが原因のセキュリティバグは**1 件もなかった**という。
- **`--no-prompt`:** Val Town は最初から `--no-prompt` で Deno を動かしていた。批判記事が問題にしていた「対話的な権限プロンプト」の状況は、そもそも起きなかった。
- **残っていた問題:** Deno 系のランタイムでも、ワーカーの中でさらにワーカーを起動するといった手口でサンドボックスを抜け出せる可能性はあった。ただし、件数も影響も vm2 よりはるかに小さかった（Val Town ブログより）。

**Deno の権限フラグの例（説明用の例）:**

```bash
# ユーザーのコードを、特定のホストへのネットワークアクセスだけ許可して実行する
deno run \
  --no-prompt \
  --allow-net=api.example.com \
  --deny-read --deny-write --deny-env --deny-run --deny-ffi \
  user-script.ts
# 許可されていない操作をすると、プロンプトを出さずに即 PermissionDenied 例外になる
```

```js
// 比較: vm2 の典型的な使い方（すでに非推奨。抜け出す方法が多数見つかっている）
const { VM } = require('vm2');
new VM({ timeout: 1000, sandbox: {} }).run(untrustedCode);
```

### ③ なぜ効果的か

- **境界の位置が違う:** vm2 は「同じ V8 コンテキストの中で JS のプロキシを使って隔離する」方式なので、言語機能の抜け穴がそのまま脱出経路になる。Deno はランタイムの**ネイティブ層**で I/O を許可リストで制御するので、JS 側の抜け穴があっても権限の外には出られない。
- **`--no-prompt`:** 権限のない操作をすると、対話なしに即座に失敗する。サーバー用途では決定的な振る舞いになり、プロンプトを悪用されることもない。
- この話は、項目 3 のグリーンスレッドが目指す「capability ベースで Isolate ごとに隔離する」設計と同じ問題意識に立っている。

---

## 9. Postgres.js 4.1：`sql.batch()` と `query_timeout`

**リンク先:** [porsager/postgres Releases](https://github.com/porsager/postgres/releases)

### ① 目的・問題意識

- 複数のクエリを順番に送ると、**クエリの数だけネットワークの往復**が発生する。
- PostgreSQL の `statement_timeout` はサーバー側のタイムアウトなので、**サーバーやネットワークが応答しなくなった場合**は検知できない。TCP がそれを検知するには数分かかることがある。
- 4.1 ではこの 2 つを解決した。Node 18 のサポートも復活した。

### ② 技術詳細

**`sql.batch([queries])`（v4.1.0 で追加）:**

- 複数のクエリを **1 回の往復**で送り、最後に Sync を 1 回だけ送る。
- トランザクションの外で使った場合、すべてのクエリがまとめてコミットされるか、まとめてロールバックされる。
- 失敗した場合はバッチ全体が reject され、失敗したクエリの位置が `error.index` に入る。
- v4.1.1 で、キャンセルしたバッチが `sql.end()` の後も `cancel_timeout` の間プロセスを生かし続けていた問題が直った。

**`query_timeout`（v4.1.0 で追加）:**

- 接続を待つ時間も含めて、クエリにかけてよい時間の上限を決める。超えるとキャンセルされる。デフォルトは無効。
- タイムアウトすると `QUERY_TIMEOUT` で reject される。
- サーバーやネットワークが応答しなくなった場合も打ち切れる。
- トランザクション制御のクエリと `forEach` にも適用される。カーソル、ストリーム、`sql.listen` には適用されない。
- 関連するオプション `cancel_timeout`（デフォルト 5 秒）は、キャンセルしたクエリが終わるのを待つ上限。超えると接続を閉じる。

**使用例（説明用の例。リリースノートには `sql.batch([queries])` というシグネチャとオプション名しか載っていない）:**

```js
import postgres from 'postgres';

const sql = postgres(process.env.DATABASE_URL, {
  query_timeout: 5_000,   // 接続待ちも含めて 5 秒で打ち切る
  cancel_timeout: 5_000,  // キャンセルが効かなければ 5 秒後に接続を閉じる
});

try {
  const [user, posts, count] = await sql.batch([
    sql`select * from users where id = ${id}`,
    sql`select * from posts where user_id = ${id} order by created_at desc limit 10`,
    sql`select count(*) from comments where user_id = ${id}`,
  ]); // 1 回の往復で送り、トランザクション外なら全部まとめてコミットかロールバック
} catch (err) {
  if (err.code === 'QUERY_TIMEOUT') console.error('タイムアウト');
  else console.error(`バッチの ${err.index} 番目のクエリが失敗`, err);
}
```

### ③ なぜ効果的か

- **バッチ化:** レイテンシの大部分を占めるのはネットワークの往復回数。それを N 回から 1 回に減らせる。特にリージョンをまたぐ DB では効果が大きい。まとめてコミット・ロールバックされるので、一貫性も簡単に保てる。
- **クライアント側のタイムアウト:** サーバー側の `statement_timeout` では検知できない障害（ネットワークの分断や、応答しなくなったサーバー）でも確実に打ち切れる。接続プールが詰まるのを防げる。

---

## 参考リンク一覧

- Node Weekly #647: https://nodeweekly.com/issues/647
- PR #67288（グリーンスレッド）: https://github.com/nodejs/node/pull/67288
- PR #65323 / #64894（Web Worker）: https://github.com/nodejs/node/pull/65323 / https://github.com/nodejs/node/pull/64894
- PR #67157 / Issue #67347（node:yaml）: https://github.com/nodejs/node/pull/67157 / https://github.com/nodejs/node/issues/67347
- X Engineering: https://blog.x.com/en/engineering/rebuilding-the-global-town-square-from-scratch
- Netlify Server: https://www.netlify.com/blog/introducing-netlify-server/
- Tom MacWright: https://macwright.com/2026/11/09/deno-security-model
- Postgres.js: https://github.com/porsager/postgres/releases
- Node.js のセキュリティリリース（解説）: https://thehackernews.com/2026/11/nodejs-fixes-16-flaws-including-http2.html
- ベンチマーク動画: https://www.youtube.com/watch?v=zDw_DxM5GlI
