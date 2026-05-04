# Node Weekly Issue 622 まとめ (April 30, 2026)

> 出典: [Node Weekly Issue 622](https://nodeweekly.com/issues/622)

---

## 1. JavaScriptの新機能総まとめ：今使えるものとNode 26で来るもの

**原文リンク:** [What's Actually New in JavaScript (And What's Coming Next) — Neciu Dan](https://neciudan.dev/whats-new-in-javascript)

### ① 目的・問題意識

ES仕様や無数のブログ記事を追いかけなくても、Node.jsで動く最新のJavaScript言語機能を一気に把握できるリファレンスとして書かれた記事。「すでに使える機能」と「Node 26で来る機能」に整理されている。

---

### ② 具体的な技術詳細

#### 2-1. Promise.try — 同期/非同期混在関数の統一ラッパー

**問題:** 同期的に例外を投げる可能性もあり、かつ Promise を返す可能性もある関数を呼び出すとき、`new Promise(r => r(fn()))` や `Promise.resolve().then(fn)` は冗長でわかりにくい。

```javascript
// ❌ 旧来パターン
function fetchData(maybeAsync) {
  return Promise.resolve()
    .then(() => maybeAsync())
    .catch(err => console.error(err));
}

// ✅ Promise.try
function fetchData(maybeAsync) {
  return Promise.try(maybeAsync)
    .catch(err => console.error(err));
}

// 実用例: 同期・非同期どちらでも安全にラップ
const result = await Promise.try(() => {
  if (Math.random() > 0.5) return "sync value";
  return fetch("/api/data").then(r => r.json()); // async
});
console.log(result);

// カウンター例
function getCount() {
  return Promise.try(() => countAsync())
    .then(result => result + 1);
}
```

- Node.js 22+ でネイティブサポート
- ES2025 正式採択 (Stage 4: 2024-10-09)

---

#### 2-2. Set の集合演算メソッド

**問題:** JavaScript の `Set` にはこれまで和集合・積集合・差集合などの演算メソッドがなく、毎回手書きのループが必要だった。

```javascript
const frontend = new Set(["JavaScript", "HTML", "CSS"]);
const backend  = new Set(["Python", "Java", "JavaScript"]);

// 和集合 (union)
const all = frontend.union(backend);
// Set { "JavaScript", "HTML", "CSS", "Python", "Java" }

// 積集合 (intersection)
const odds    = new Set([1, 3, 5, 7, 9]);
const squares = new Set([1, 4, 9]);
console.log(odds.intersection(squares)); // Set { 1, 9 }

// 差集合 (difference)
console.log(odds.difference(squares)); // Set { 3, 5, 7 }

// 対称差 (symmetricDifference)
console.log(odds.symmetricDifference(squares)); // Set { 3, 4, 5, 7 }

// 部分集合チェック (isSubsetOf)
const declarative = new Set(["HTML", "CSS"]);
console.log(declarative.isSubsetOf(frontend)); // true

// 上位集合チェック (isSupersetOf)
console.log(frontend.isSupersetOf(declarative)); // true

// 互いに素チェック (isDisjointFrom)
console.log(odds.isDisjointFrom(new Set([2, 4, 6]))); // true
```

- Node.js 22+ / Chrome 122+ / Firefox 127+ でサポート済み

---

#### 2-3. Array.fromAsync — 非同期イテラブルから配列へ

**問題:** 非同期イテレータの結果を配列にするには `for-await-of` ループと push が必要で冗長だった。

```javascript
// ❌ 旧来パターン
async function collectStream(stream) {
  const chunks = [];
  for await (const chunk of stream) {
    chunks.push(chunk);
  }
  return chunks;
}

// ✅ Array.fromAsync
async function collectStream(stream) {
  return Array.fromAsync(stream);
}

// 非同期ジェネレータとの組み合わせ
async function* generateNumbers() {
  yield 1;
  yield 2;
  yield 3;
}
const result = await Array.fromAsync(generateNumbers());
console.log(result); // [1, 2, 3]

// Node.js glob との実用例
import { glob } from 'node:fs/promises';
const files = await Array.fromAsync(glob('./src/**/*.ts'));

// マッピング関数付き
const doubled = await Array.fromAsync(
  generateNumbers(),
  x => x * 2
);
console.log(doubled); // [2, 4, 6]
```

- `Promise.all` と異なり、値を **順次取得** (前の値が settle してから次を取得)
- Node.js 22+ でサポート済み

---

#### 2-4. using / await using — 明示的リソース管理

**問題:** ファイルハンドル・DBコネクション・ロックなどのリソースを `try-finally` で確実に解放するのは冗長でバグが混入しやすかった。

```javascript
// ❌ 旧来パターン
let file;
try {
  file = await fs.open('data.txt', 'r');
  const content = await file.read();
  return content;
} finally {
  await file?.close();
}

// ✅ await using (非同期リソース管理)
import fs from 'node:fs/promises';

async function readFile() {
  // スコープを抜けると自動的に file[Symbol.asyncDispose]() が呼ばれる
  await using file = await fs.open('data.txt', 'r');
  const { buffer } = await file.read({ buffer: Buffer.alloc(1024) });
  return buffer;
}

// ✅ using (同期リソース管理)
function processWithLock() {
  using lock = acquireLock();  // スコープ抜けで lock[Symbol.dispose]() が自動呼出し
  // ... 処理 ...
} // ← ここで自動解放

// カスタム disposable オブジェクトの作成
const connection = {
  query: (sql) => db.execute(sql),
  [Symbol.asyncDispose]: async () => {
    await db.release();
    console.log('Connection released');
  }
};

async function doWork() {
  await using conn = connection;
  return conn.query('SELECT 1');
} // ← 自動で release()
```

- Node.js 20.9+ / Chrome 123+ / Firefox 119+ でサポート済み

---

#### 2-5. Math.sumPrecise — 精密な浮動小数点合計（Node 26）

**問題:** JavaScript では浮動小数点の中間演算で精度損失が蓄積する。`[1e20, 0.1, -1e20].reduce((a, b) => a + b)` は `0` になってしまう。

```javascript
// ❌ 従来の reduce によるループ合計
const values = [1e20, 0.1, -1e20];
console.log(values.reduce((a, b) => a + b)); // 0 ← 精度損失！

// ✅ Math.sumPrecise
console.log(Math.sumPrecise([1e20, 0.1, -1e20])); // 0.1 ← 正確！

// 基本的な例
console.log(Math.sumPrecise([1, 2]));         // 3
console.log(Math.sumPrecise([0.1, 0.2]));     // 0.30000000000000004
// ※ 0.1 + 0.2 問題は浮動小数点リテラル自体の精度限界なので解決しない

// 金融計算での実用例
const transactions = [
  100000000000000.0,
  0.001,
  -100000000000000.0
];
console.log(Math.sumPrecise(transactions)); // 0.001（正確）
```

- 数学的精密値でそれぞれの数を扱い、最後に64ビット浮動小数点に変換
- Baseline 2026 (2026年4月〜 全主要ブラウザ対応)
- Node 26 (V8 14.6) で利用可能

---

#### 2-6. Map.getOrInsert / getOrInsertComputed — Mapのupsert（Node 26）

**問題:** `Map` で「キーがあれば取得、なければデフォルト値を挿入して返す」パターンは毎回 `has()` + `get()` + `set()` の3アクションが必要だった。

```javascript
// ❌ 旧来パターン
const counts = new Map();
for (const word of words) {
  if (!counts.has(word)) counts.set(word, 0);
  counts.set(word, counts.get(word) + 1);
}

// ✅ getOrInsert
const counts = new Map();
for (const word of words) {
  counts.set(word, counts.getOrInsert(word, 0) + 1);
}

// デフォルト設定
const options = readConfig();
options.getOrInsert("theme", "light");
options.getOrInsert("fontSize", 14);

// グルーピング
const grouped = new Map();
for (const [key, ...values] of data) {
  grouped.getOrInsert(key, []).push(...values);
}

// ✅ getOrInsertComputed (高コストなデフォルト値向け)
// ← キーが存在するときはコールバックを呼ばない
const cache = new Map();
function getUser(id) {
  return cache.getOrInsertComputed(id, () => fetchUserFromDB(id));
}

// WeakMap でも使える
const meta = new WeakMap();
meta.getOrInsertComputed(obj, () => ({ created: Date.now() }));
```

- Node 26 (V8 14.6) で利用可能

---

#### 2-7. Iterator.concat — イテレータの連結（Node 26）

**問題:** 複数のイテレータを順に連結するにはジェネレータ関数と `yield*` が必要で冗長だった。

```javascript
// ❌ ジェネレータを使った旧来パターン
function* concat(a, b) {
  yield* a;
  yield* b;
}

// ✅ Iterator.concat
const lows  = Iterator.from([0, 1, 2, 3]);
const highs = Iterator.from([6, 7, 8, 9]);

const all = Iterator.concat(lows, highs);
console.log([...all]); // [0, 1, 2, 3, 6, 7, 8, 9]

// 中間値の挿入も可能
const digits = Iterator.concat(
  Iterator.from([0, 1, 2, 3]),
  [4, 5],                         // 通常の配列も可
  Iterator.from([6, 7, 8, 9])
);
console.log([...digits]); // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

- Node 26 (V8 14.6) で利用可能

---

#### 2-8. import defer — モジュール評価の遅延

**問題:** ESM では `import` した時点でモジュールが評価されるが、起動時に不要なモジュールも即座に実行されパフォーマンスに影響する。

```javascript
// ✅ import defer: モジュールを「ダウンロードはするが評価しない」
import defer * as print from './print.js';

// ← ここでは print モジュールは評価されていない

button.addEventListener('click', () => {
  // 初めてプロパティにアクセスした時点で評価が実行される
  print.render(document);
});

// webpack での動的ロケール例
const locale = import.defer('./locales/' + language + '.js');
// ← 全ロケールのネットワークリクエストは発生するが評価はしない

button.onclick = () => {
  console.log(locale.greeting); // ← ここで初めて評価
};
```

---

### ③ なぜ効果的か

- これらの機能はすべて **Node.js ランタイムにネイティブ実装済み**（polyfill不要）
- `using`/`await using` により **リソースリークバグを構造的に排除** できる
- `Promise.try` で **同期・非同期混在コードの統一的なエラーハンドリング** が可能
- Set演算・`Array.fromAsync`・`Map.getOrInsert` はよくある冗長パターンを **標準APIで置換**
- `Math.sumPrecise` で **金融・科学計算の中間誤差蓄積** を回避できる

---

## 2. Node.js 26.0: Temporal が Rosetta 2 でつまずき、リリースが5月5日へ延期

**原文リンク:** [Node 26.0 Slips to May 5: Temporal Trips Over Rosetta 2 (GitHub PR #62526)](https://github.com/nodejs/node/pull/62526)

### ① 目的・問題意識

Node.js 26.0 (Current) は2026年4月28日のリリースを予定していたが、macOS の Rosetta 2（Apple Silicon 上でx86バイナリを動かす互換レイヤー）環境でのビルドが壊れたため5月5日に延期された。原因は Temporal API の V8 実装が内部で使う Rust コンポーネント（Unicode カレンダー処理）が Rosetta 2 下でコンパイルエラーになるというもの。

### ② 技術詳細：Node 26 の主な変更

#### Temporal API のデフォルト有効化

```javascript
// Node 26 以降、import/require なしで直接利用可能
const today = Temporal.Now.plainDateISO();
console.log(today.toString()); // "2026-05-05"

const meeting = Temporal.ZonedDateTime.from(
  '2026-05-05T14:00:00[America/New_York]'
);
const tokyoMeeting = meeting.withTimeZone('Asia/Tokyo');
console.log(tokyoMeeting.toString());
// "2026-05-06T03:00:00+09:00[Asia/Tokyo]"
```

#### V8 14.6 への更新（Chromium 146）

新たに含まれる機能:
- `Map.prototype.getOrInsert()` / `getOrInsertComputed()`
- `WeakMap.prototype.getOrInsert()` / `getOrInsertComputed()`
- `Iterator.concat()` サポート

#### 主な破壊的変更 (Semver-Major)

```javascript
// ❌ 削除: http.Server.prototype.writeHeader()
// ✅ 代替: writeHead() を使うこと
response.writeHead(200, { 'Content-Type': 'text/plain' });

// ❌ 削除: レガシーストリームモジュール
// require('_stream_wrap')         // 削除
// require('_stream_readable')     // 削除
// require('_stream_writable')     // 削除
// require('_stream_duplex')       // 削除
// require('_stream_transform')    // 削除
// require('_stream_passthrough')  // 削除

// ❌ 削除: --experimental-transform-types フラグ
// ✅ 代替: --strip-types / --transform-types (安定版)

// ⚠️ Runtime Deprecated: module.register()
// ⚠️ DEP0203, DEP0204 (crypto APIs) → Runtime deprecation に昇格
```

#### その他の更新

| 項目 | 変更内容 |
|------|---------|
| Undici | 8.0.2 に更新（HTTP クライアント） |
| NODE_MODULE_VERSION | 147（ネイティブアドオンの再ビルドが必要） |
| GCC 要件 | 13.2 以上に引き上げ |
| Python サポート | 3.9 を削除 |
| Windows SDK | バージョン 11 が必要 |

### ③ なぜ効果的か

- **Temporal のデフォルト有効化** により、Node.js 26 からはpolyfillなしで `Date` の後継を利用できる
- `Map.getOrInsert` / `Iterator.concat` も V8 14.6 経由で自動的に利用可能
- 破壊的変更の整理により技術的負債が解消され、メンテナビリティが向上

---

## 3. pnpm 11.0 リリース — SQLite ストア・SEA対応・サプライチェーン保護

**原文リンク:** [pnpm 11.0 Release Notes](https://pnpm.io/blog/releases/11.0) / [InfoQ記事](https://www.infoq.com/news/2026/04/pnpm-11-rc-release/)

### ① 目的・問題意識

pnpm 11 は以下の問題を解決する:
- ストアインデックスがファイル数千万個のJSONファイルで構成されていた（I/Oコスト大）
- グローバルインストールが相互に干渉してピア依存コンフリクトが発生していた
- サプライチェーン攻撃対策がオプトインだった

### ② 技術詳細

#### SQLite ベースのパッケージストア

```bash
# 旧: $STORE/index/ 以下に数百万個のJSONファイル
# 新: $STORE/index.db (単一のSQLiteファイル、WALモード)

# MessagePack値を使ったスキーマ、並行アクセス対応
```

#### pack-app コマンド（Single Executable Applications）

```bash
# CommonJS エントリファイルを単一実行バイナリにパッケージ化
pnpm pack-app src/main.cjs

# プラットフォーム指定
pnpm pack-app src/main.cjs --target linux-x64
pnpm pack-app src/main.cjs --target win32-x64
pnpm pack-app src/main.cjs --target darwin-arm64

# package.json での設定
{
  "pnpm": {
    "app": {
      "entry": "src/main.cjs",
      "targets": ["linux-x64", "darwin-arm64"]
    }
  }
}
```

内部的に Node.js の Single Executable Applications (SEA) API を使用。Node 25.5+ の `--build-sea` フラグを活用。

#### 隔離されたグローバルインストール

```bash
# 各グローバルパッケージが独立した仮想ストアに配置される
pnpm add -g typescript
# -> {pnpmHomeDir}/global/v11/{hash}/
#      package.json
#      node_modules/
#      lockfile

# パッケージ同士のピア依存コンフリクトが発生しなくなる
pnpm add -g prettier  # typescript とは完全隔離
```

#### サプライチェーン保護のデフォルト有効化

```jsonc
// .npmrc または package.json の pnpm セクション
// 以下がデフォルト値として自動適用される:
{
  "pnpm": {
    "minimumReleaseAge": 1440,   // 新規パッケージは公開後1日経過後のみインストール可
    "blockExoticSubdeps": true   // 不審な依存関係パターンをブロック
  }
}
```

#### システム要件の変更

- **Node.js 22以上が必要**（pnpm 自体が純粋な ESM に移行）
- Node.js 18/20 のサポートを終了

### ③ なぜ効果的か

- **SQLite ストア** により大規模プロジェクトでのインストール・検索速度が向上
- **隔離グローバル** でCLIツール同士の干渉バグが根絶される
- **サプライチェーン保護デフォルト** により設定不要でセキュアな環境を提供
- **pack-app** で追加ツール不要でNode.jsアプリを配布可能な単一バイナリにできる

---

## 4. Perry — Rustで書かれたTypeScript→ネイティブバイナリコンパイラ

**原文リンク:** [Perry — TypeScript → Native](https://www.perryts.com/) / [GitHub: PerryTS/perry](https://github.com/PerryTS/perry)

### ① 目的・問題意識

TypeScript をデプロイするには Node.js や Bun などのランタイムが必要で、起動が遅く・バイナリサイズが大きく・依存関係の管理が必要だった。Perry はTypeScriptをSWC+LLVMで直接ネイティブバイナリにコンパイルし、ランタイム依存をなくす。

### ② 技術詳細

#### コンパイルパイプライン

```
TypeScript → (SWC) → HIR → (最適化) → LLVM IR → ネイティブバイナリ
```

1. **SWC** でTypeScriptを解析
2. **HIR (High-level IR)** に変換
3. クロージャ変換・async処理・インライン化などの最適化パス
4. **LLVM** でネイティブマシンコードを生成
5. システムCリンカーで最終バイナリを生成

#### インストールと使い方

```bash
# npm 経由
npm install @perryts/perry
npx perry compile src/main.ts -o myapp

# macOS (Homebrew)
brew install perryts/perry/perry

# 実行
./myapp  # Node.js 不要！
```

#### コード例：Fastify サーバー

```typescript
import fastify from 'fastify';
import { config } from './config';

const app = fastify();

app.get('/api/users', async () => {
  return [{ id: 1, name: 'Alice' }];
});

app.listen({ port: config.port }, () => {
  console.log(`Server running on port ${config.port}`);
});
```

```bash
# コンパイルして実行
perry compile src/main.ts -o my-api
./my-api  # ランタイム不要
```

#### パフォーマンス比較（Node.js v25 対比）

| ベンチマーク | Perry | Node.js | 倍率 |
|-----------|-------|---------|------|
| factorial | 31ms | 596ms | **19×** |
| closure | 10ms | 309ms | **31×** |
| fibonacci(40) | 320ms | 1033ms | **3.2×** |
| method_calls | 1ms | 11ms | **11×** |

#### バイナリサイズ

| 内容 | サイズ |
|------|--------|
| Hello world | ~330 KB |
| fs/path/process 込み | ~380 KB |
| フルstdlib アプリ | ~48 MB |

#### クロスプラットフォームターゲット

```bash
perry compile src/main.ts --target darwin-arm64
perry compile src/main.ts --target linux-x64
perry compile src/main.ts --target win32-x64
# モバイル・WebAssembly・ネイティブUI も対応
```

#### マルチスレッド

```typescript
import { parallelMap, parallelFilter, spawn } from 'perry/thread';

const results = parallelMap([1, 2, 3, 4, 5], n => fibonacci(n));
const evens = parallelFilter(numbers, n => n % 2 === 0);
const result = await spawn(() => expensiveComputation());
```

### ③ なぜ効果的か

- JITウォームアップが不要なため**起動速度が大幅向上**
- Node.js/Bun を本番サーバーに入れる必要がなく**デプロイが簡素化**
- LLVM の自動ベクトル化（fast-math フラグ）により数値計算ベンチマークでC++/Rustと競合できる速度
- `npm install @perryts/perry` でインストールでき**既存TypeScriptコードを最小変更で移行可能**

---

## 5. portless — ポート番号をなくして開発サーバーに名前付きURLを

**原文リンク:** [GitHub: vercel-labs/portless](https://github.com/vercel-labs/portless)

### ① 目的・問題意識

`localhost:3000`・`localhost:3001`・`localhost:8080` などのポート番号は覚えにくく、チームで開発する際に衝突する。AIエージェントがlocalhostサーバーにアクセスする際もポート番号の管理が課題。portless は名前付きの `.localhost` URLで開発サーバーを管理する。

### ② 技術詳細

#### 基本的な使い方

```bash
# インストール
npm install -g portless  # または pnpm/bun

# 使い方: portless <名前> <コマンド>
portless myapp next dev
# -> https://myapp.localhost にアクセス可能

# ポート番号は内部で自動割り当て (4000-4999)
```

#### 設定ファイル (portless.json)

```json
{ "name": "myapp" }
```

```bash
# 設定ファイルがあれば名前なしで実行可能
portless
```

#### モノレポ対応

```json
{
  "apps": {
    "apps/web": { "name": "myapp" },
    "apps/api": { "name": "api.myapp" }
  }
}
```

#### サブドメインでサービスを整理

```bash
portless api.myapp pnpm start
# -> https://api.myapp.localhost
# Webアプリ: https://myapp.localhost
# APIサーバー: https://api.myapp.localhost
```

#### Gitワークツリー対応

```bash
# main ブランチ
portless run next dev   # -> https://myapp.localhost

# fix-ui ブランチ (worktreeで別ディレクトリ)
portless run next dev   # -> https://fix-ui.myapp.localhost
# ブランチ名が自動でサブドメインプレフィックスになる
```

#### Tailscale 統合（新機能）

```bash
# チームのTailscaleネットワーク越しに共有
portless myapp --tailscale next dev
# ローカル: https://myapp.localhost
# Tailnetメンバー: https://devbox.yourteam.ts.net

# 公開インターネットへの公開 (Tailscale Funnel)
portless myapp --funnel next dev
```

#### アーキテクチャ

```
[ブラウザ] -> https://myapp.localhost:443
                   ↓ (ローカルCA証明書でHTTPS)
           [portless プロキシ :80/:443]
                   ↓ (HTTP/2多重化)
           [アプリ :ランダムポート]
```

- HTTPS + HTTP/2 がデフォルト有効（初回起動時にローカルCA証明書を生成・信頼）
- フレームワーク自動検出（Vite、Astro、React Router 等の `--port` フラグを自動挿入）

### ③ なぜ効果的か

- **ポート衝突がなくなる**（エフェメラルポートを自動割り当て）
- **HTTPS デフォルト** でブラウザ警告なしにService Worker等のセキュアAPIをテストできる
- **Gitワークツリー対応** により複数ブランチを同時に動かしてもURLが衝突しない
- **Tailscale統合** でVPN越しにチームメンバーと開発環境を即座に共有できる

---

## 6. Fresh 2.3 — デフォルトゼロJS・WebSocket・View Transitions

**原文リンク:** [Fresh 2.3: Zero JS by default, View Transitions, and Temporal support (Deno)](https://deno.com/blog/fresh-2.3)

### ① 目的・問題意識

DenoのWebフレームワークFreshは「Islands Architecture」でインタラクティブな部分だけJSを送信する思想だが、静的ページでも不要なJSが送られていた。2.3 では純粋静的ページは完全にJSゼロになり、画面遷移も View Transitions API でスムーズになる。

### ② 技術詳細

#### ゼロJS by default（静的ページ）

```typescript
// routes/index.tsx — 静的ページ（Islandなし）
// Fresh 2.3 からはこのページにJSは一切送信されない
export default function Home() {
  return (
    <main>
      <h1>Hello, World!</h1>
      <p>このページはJSゼロで配信されます</p>
    </main>
  );
}
```

#### WebSocket サポート

```typescript
// routes/ws.ts — WebSocketエンドポイント
import { define } from "../utils.ts";

export const handler = define.handlers({
  GET(ctx) {
    const { socket, response } = ctx.upgrade();
    // 非WebSocketリクエストは自動的に400を返す

    const clients = new Set<WebSocket>();

    socket.onopen = () => {
      clients.add(socket);
      console.log("Client connected");
    };

    socket.onmessage = (e) => {
      // 全クライアントにブロードキャスト
      for (const client of clients) {
        if (client.readyState === WebSocket.OPEN) {
          client.send(e.data);
        }
      }
    };

    socket.onclose = () => {
      clients.delete(socket);
    };

    return response;
  }
});
```

#### View Transitions（ページ遷移アニメーション）

```typescript
// routes/_app.tsx に1つの属性を追加するだけ
export default function App({ Component }: PageProps) {
  return (
    <html>
      <head>
        <meta name="fresh-partial" content="true" />
        {/* View Transitions を有効化 */}
        <meta name="fresh-view-transitions" content="true" />
      </head>
      <body>
        <Component />
      </body>
    </html>
  );
}

// Fresh の Partials システムと連携し、
// ブラウザが対応していない場合は通常の部分更新にフォールバック
```

#### Temporal API の Island Props 対応

```typescript
// サーバーサイド (routes/countdown.tsx)
import { Temporal } from "@js-temporal/polyfill";

export default function CountdownPage() {
  const target = Temporal.PlainDate.from("2026-12-31");
  // Temporal オブジェクトをIslandに渡せる
  return <Countdown target={target} />;
}

// islands/Countdown.tsx
interface Props {
  target: Temporal.PlainDate;  // 8つのTemporal型すべてをpropsで受け取り可能
}

export default function Countdown({ target }: Props) {
  const today = Temporal.Now.plainDateISO();
  const diff = today.until(target);
  return <p>あと {diff.days} 日</p>;
}
```

対応Temporal型: `Instant`, `ZonedDateTime`, `PlainDate`, `PlainTime`, `PlainDateTime`, `PlainYearMonth`, `PlainMonthDay`, `Duration`

### ③ なぜ効果的か

- **ゼロJS** によりLighthouse/Core Web Vitalsスコアが静的ページで完璧になる
- **WebSocket組み込み** でサードパーティライブラリなしにリアルタイム機能を実装できる
- **View Transitions** でSPAライクなスムーズ遷移をサーバーサイドフレームワークで実現
- **Temporal Island Props** により日時データを安全に型情報付きでIslandに渡せる

---

## 7. Honker — SQLiteにPostgreSQL風NOTIFY/LISTENを追加

**原文リンク:** [GitHub: russellromney/honker](https://github.com/russellromney/honker) / [Show HN: Hacker News](https://news.ycombinator.com/item?id=47874647)

### ① 目的・問題意識

SQLite はシンプルで高速だが、単一ファイルの設計上「イベント通知」機能がなく、リアルタイムの変更検知にはポーリングが必要だった。また、タスクキューを実装するために Redis や別のブローカーを導入するのはオーバーヘッドが大きかった。Honker はSQLiteのエクステンションとして Postgres スタイルの `NOTIFY`/`LISTEN` セマンティクスを追加する。

### ② 技術詳細

#### 動作原理

```
[トランザクション内のINSERT] -> [PRAGMA data_version が増加]
                                    ↓ (1ms間隔でポーリング)
                              [サブスクライバーに通知]
                              (クロスプロセス遅延: ~0.7ms p50)
```

`PRAGMA data_version` はSQLiteがすべてのコミット時にインクリメントするカウンタ。Honker はこれを1msごとに監視することで、ブローカーデーモンなしにプッシュ通知的なセマンティクスを実現。

#### 3つのプリミティブ

**1. notify() — エフェメラルPub/Sub（永続化なし）**

```javascript
const { open } = require('@russellthehippo/honker-node');
const db = open('app.db');

// パブリッシュ（ビジネスロジックと同一トランザクションで原子的）
const tx = db.transaction();
tx.execute('INSERT INTO orders (id) VALUES (?)', [42]);
tx.notify('orders', { id: 42 });  // ← 同じトランザクション内
tx.commit();  // ロールバックすれば通知も消える

// サブスクライブ
db.listen('orders', (payload) => {
  console.log('New order:', payload.id);
});
```

**2. stream() — 永続Pub/Sub（コンシューマーオフセット付き）**

```javascript
// パブリッシュ
const stream = db.stream('user-events');
const tx = db.transaction();
tx.execute('UPDATE users SET name = ? WHERE id = ?', ['Bob', 42]);
stream.publish({ user_id: 42, change: 'name' }, tx);
tx.commit();

// サブスクライブ（オフセットは自動管理、再起動後も続きから再生）
for await (const event of stream.subscribe('dashboard-consumer')) {
  await pushToBrowser(event);
}
```

**3. queue() — 最低一回保証のワークキュー**

```javascript
const emails = db.queue('emails');

// エンキュー（ビジネスロジックと原子的）
const tx = db.transaction();
tx.execute('INSERT INTO orders (id) VALUES (?)', [42]);
emails.enqueue({ to: 'alice@example.com', subject: '注文確認' }, tx);
tx.commit();  // ロールバックでジョブも消える

// ワーカー
const job = await emails.claim('worker-1');
try {
  await sendEmail(job.payload);
  job.ack();  // 処理完了
} catch (err) {
  job.nack(); // リトライキューに戻す
}
```

#### Node.js パッケージ

```bash
npm install @russellthehippo/honker-node
```

Python, Rust, Go, Ruby, Bun, Elixir 向けのバインディングも提供。

### ③ なぜ効果的か

- **ブローカーデーモン不要** で Redis/RabbitMQ等の追加インフラが不要
- **ビジネスロジックと通知/ジョブが同一トランザクション** でアトミックに処理できる（DB書き込みが成功したらジョブが必ず実行される）
- **0.7ms p50 のクロスプロセス遅延** でほぼリアルタイムの通知が可能
- SQLiteの単一ファイル設計を維持したまま**本番レベルのメッセージング機能**を追加できる

---

## 8. AVA 8.0 — 完全ESM移行と条件付きテストモディファイア

**原文リンク:** [GitHub: avajs/ava Releases](https://github.com/avajs/ava/releases)

### ① 目的・問題意識

AVA はNode.jsの並行テストランナーだが、CommonJSでの設計が残っておりESMプロジェクトとの相性に課題があった。8.0で完全にESMに移行し、条件付きスキップ/実行モディファイアを追加することでCIマトリックスやOS依存テストの管理を改善する。

### ② 技術詳細

#### 完全ESM化

```javascript
// ava.config.mjs (設定ファイルもESM)
export default {
  files: ['tests/**/*.test.js'],
  // すべてのテストファイルが import() で読み込まれる
  // 対応拡張子: .js, .mjs (デフォルト)
};
```

#### test.skipIf() と test.runIf() — 条件付きテスト制御

```javascript
import test from 'ava';

// 条件が true のときテストをスキップ
test.skipIf(process.platform === 'win32')(
  'Windowsでは動かない機能',
  t => {
    t.pass();
  }
);

// 条件が true のときのみテストを実行（skipIf の逆）
test.runIf(process.platform === 'linux')(
  'Linuxのみで実行',
  t => {
    t.pass();
  }
);

// 他のモディファイアとの組み合わせ
test.serial.skipIf(process.env.CI === 'true')(
  'ローカルのみシリアル実行',
  async t => {
    await heavyOperation();
    t.pass();
  }
);

test.failing.runIf(process.env.EXPERIMENTAL === 'true')(
  '実験的機能のフェイリングテスト',
  t => {
    t.throws(() => experimentalApi());
  }
);
```

### ③ なぜ効果的か

- **完全ESM** でTypeScript/ESMプロジェクトとの設定不要な統合が可能
- **skipIf/runIf** でOS・環境変数・CIフラグに応じたテスト制御をコード内で宣言的に表現できる
- 従来は `if (process.platform !== 'linux') return;` のような命令的な早期リターンが必要だったが、**モディファイアで意図が明確**になる

---

## ブリーフニュース

### Eleventy (11ty) Pro Edition

静的サイトジェネレーターの Eleventy がPro Editionのクラウドファンディングを開始。協調編集機能と簡易パブリッシング機能を追加予定。

### そのほかのリリース

| パッケージ | バージョン | 概要 |
|-----------|-----------|------|
| Hono Node.js Adapter | 2.0 | Node.js向けHonoアダプター |
| BWIP-JS | 4.10 | バーコード生成ライブラリ |
| basic-ftp | 6.0 | FTPクライアント |
| icc | 4.0 | ICCカラープロファイル解析 |
| Ora | 9.4 | ターミナルスピナー |
| Mongoose | 9.6 | MongoDB ODM |
| Vine | 4.4 | フォームバリデーション |

---

## まとめ

| トピック | 重要度 | アクション |
|---------|--------|-----------|
| JavaScriptの新機能 (Promise.try, Set演算, Array.fromAsync, using) | ★★★ | 既存コードのボイラープレートを置き換える |
| Node.js 26 (Temporal デフォルト有効, V8 14.6) | ★★★ | 5月5日リリース後、Node 26への移行計画を立てる |
| pnpm 11.0 (SQLiteストア, SEA, サプライチェーン保護) | ★★☆ | Node.js 22に移行済みなら pnpm 11 へアップグレード |
| Perry (TypeScript→ネイティブバイナリ) | ★★☆ | CLIツール・バックエンドサービスの配布で試す |
| portless (名前付きローカルURL) | ★★☆ | 複数サービスを同時開発する場合に導入を検討 |
| Fresh 2.3 (ゼロJS, WebSocket, View Transitions) | ★★☆ | DenoベースのSSRフレームワークを使っている場合は更新 |
| Honker (SQLite NOTIFY/LISTEN) | ★★☆ | SQLiteアプリにタスクキューやPub/Subが必要なら検討 |
| AVA 8.0 (完全ESM, skipIf/runIf) | ★☆☆ | AVAユーザーは8.0にアップグレード |
