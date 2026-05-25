# Node Weekly Issue 624 (May 14, 2026)

**タイトル: "Let's go deep into Node's internals"**
**URL:** https://nodeweekly.com/issues/624

---

## 特集: NodeBook — An Advanced Guide to Node.js Internals

**URL:** https://www.thenodebook.com/

Volume I が全8章・46サブチャプターで完成。Volume II（ネットワーク I/O と API）は制作中。  
Node.js を使って開発しているが、より深い動作原理を理解したい開発者向けに書かれた、無料で読めるオープンガイド。

---

## 1. Node.js とは何か — ランタイムアーキテクチャ、V8 & libuv

**URL:** https://www.thenodebook.com/node-arch/what-is-nodejs

### ① 目的・問題意識

Node.js が「シングルスレッドなのに高性能」な理由を、V8・libuv・Node core の3層から説明する。

### ② 技術詳細

ランタイムは4つの表面から構成される。

| コンポーネント | 役割 |
|---|---|
| **V8** | JavaScript のパース・コンパイル・最適化・実行・ヒープ管理 |
| **libuv** | イベントループ実行、ハンドル/リクエスト追跡、クロスプラットフォーム I/O、ワーカープール管理 |
| **Node Core APIs** | `node:fs`, `node:http`, `node:net`, `node:crypto`, `node:stream`, `node:process` などの JavaScript モジュール公開 |
| **パッケージエコシステム** | npm レジストリとサードパーティライブラリ群 |

**同時実行モデル（concurrency model）**
- JavaScript は**1つのメインスレッド**上で動作する
- 遅いネイティブ処理（ファイル読み込みなど）は libuv のスレッドプールに委ねられる
- ネイティブ側が結果を返すと Node が対応する JavaScript コールバック・Promise リアクションをスケジュールする

**スレッドプール**
```bash
# デフォルトはスレッド4本。UV_THREADPOOL_SIZE で変更可能
UV_THREADPOOL_SIZE=8 node server.js
```

`fs.readFile` を呼ぶと：
1. バインディングが libuv にリクエストを渡す
2. libuv がスレッドプールでファイルを読む
3. 読み終わったら「完了イベント」をキューに積む
4. イベントループが完了イベントを検出し、C++ バインディング経由で JS コールバックを呼ぶ

**OS レベルの I/O 多重化**
| OS | 仕組み |
|---|---|
| Linux | epoll |
| macOS | kqueue |
| Windows | IOCP (I/O Completion Ports) |

libuv がこれらの差異を吸収し、単一の C API を提供する。

### ③ なぜ効果的か

「シングルスレッドに見えるが I/O はノンブロッキング」という Node の設計哲学が、Nginx のようなイベント駆動サーバーと同様の高並列処理を可能にする根拠となる。

---

## 2. V8 エンジン — JIT、隠しクラス、逆最適化

**URL:** https://www.thenodebook.com/node-arch/v8-engine-intro

### ① 目的・問題意識

V8 の JIT パイプラインを理解することで、パフォーマンスを損なうコードパターンを避け、最適化コードを書けるようにする。

### ② 技術詳細

**JIT コンパイルパイプライン**

```
JavaScript ソース
      ↓ パース
 Ignition（バイトコード生成）
      ↓ 型フィードバック収集（ホットパス検出）
 Sparkplug（ベースラインコンパイル）
      ↓
 Maglev（中間最適化）
      ↓
 TurboFan（フル最適化・機械語生成）
```

**隠しクラス（Hidden Classes）**

V8 はオブジェクトの「形状（shape）」をキャッシュする。同じプロパティ順のオブジェクトは同じ隠しクラスを共有し、インラインキャッシュ（IC）によって高速にアクセスできる。

```javascript
// ✅ 良い例: 同じ形状（隠しクラスが安定）
function Point(x, y) {
  this.x = x;
  this.y = y;
}
const p1 = new Point(1, 2);
const p2 = new Point(3, 4);

// ❌ 悪い例: 動的にプロパティ形状が変わる
const obj = { x: 1 };
obj.y = 2;  // 新しい隠しクラスに遷移
delete obj.x;  // さらに別の隠しクラスに遷移 → IC の無効化
```

**逆最適化（Deoptimization）**

TurboFan が「型が安定」と仮定して最適化した後、実行時にその仮定が崩れると V8 は最適化コードを即座に破棄し、Maglev/Sparkplug/Ignition へ降格する。これが繰り返されると「最適化 → 逆最適化のループ」でパフォーマンスが大幅に劣化する。

```bash
# 逆最適化を追跡するフラグ
node --trace-deopt app.js
```

**パフォーマンス改善のためのベストプラクティス**

| アンチパターン | 理由 |
|---|---|
| `delete obj.prop` | 隠しクラスを変える |
| 同一関数に異なる型を渡す（多態性） | IC が megamorphic になる |
| `arguments` オブジェクトの使用 | 最適化を阻害 |
| 動的プロパティ追加 | 形状が不安定になる |

### ③ なぜ効果的か

安定したオブジェクト形状を保つことで、V8 は IC とコンパイル済み最適化コードを使い回せるようになり、CPU 実行効率が大幅に向上する。

---

## 3. イベントループ詳解

**URL:** https://www.thenodebook.com/node-arch/event-loop-intro

### ① 目的・問題意識

イベントループの各フェーズを正確に理解し、`setTimeout` / `setImmediate` / `process.nextTick` / Promise の実行順序を予測できるようにする。

### ② 技術詳細

**イベントループの6フェーズ（libuv 管理）**

```
timers          → setTimeout / setInterval コールバック
pending callbacks → 前のループで保留された I/O エラーコールバック
idle/prepare    → 内部用
poll            → 新しい I/O イベント待機・コールバック実行
check           → setImmediate コールバック
close callbacks → socket.on('close', ...) など
```

**Poll フェーズ（最重要）**

- 完了した I/O イベントのコールバックを実行
- 次のタイマーまでの待機時間を計算してブロック（CPU 使用なしで待機）
- OS（epoll/kqueue/IOCP）が I/O 完了を通知するまで待つ

**ミクロタスクキュー（2種類）**

各フェーズのコールバック実行後に必ず全ドレインされる。

1. `process.nextTick` キュー（Node 独自、優先度が高い）
2. Promise ミクロタスクキュー（V8 管理、`PromiseReactionJob`）

```javascript
setImmediate(() => console.log('setImmediate'));
setTimeout(() => console.log('setTimeout'), 0);
Promise.resolve().then(() => console.log('Promise'));
process.nextTick(() => console.log('nextTick'));

// 出力順:
// nextTick          ← 最優先（nextTick キュー）
// Promise           ← ミクロタスクキュー
// setTimeout        ← timers フェーズ（またはsetImmediateの後）
// setImmediate      ← check フェーズ
```

**I/O コールバック内での実行順序**

```javascript
const fs = require('fs');
fs.readFile('file.txt', () => {
  // poll フェーズ内で実行中
  setTimeout(() => console.log('setTimeout'), 0);
  setImmediate(() => console.log('setImmediate'));
  // setImmediate が必ず先。poll の次は check フェーズのため
});
```

### ③ なぜ効果的か

フェーズを正確に理解することで、`setImmediate` と `setTimeout(fn, 0)` の実行順序の不確実性（I/O コールバック外では非決定論的）を把握し、意図した順序でコードを動かせる。

---

## 4. Buffer の基礎 — バイト列、エンコーディング、TypedArray メモリ

**URL:** https://www.thenodebook.com/buffers/what-is-buffer

### ① 目的・問題意識

Buffer が何者で、TypedArray とどう違い、V8 ヒープの「外側」に存在するメモリをどう扱うかを理解する。

### ② 技術詳細

**Buffer と TypedArray の関係**

Node.js v3.0 以降、`Buffer` は `Uint8Array` の直接サブクラス。ただし重要な違いがある：

```javascript
// Buffer.prototype.slice() → 同一メモリへのビュー（コピーなし）
const source = Buffer.from('abcd');
const view = source.slice(1, 3);  // 'bc' の参照

// TypedArray.prototype.slice() → コピーを作成
const arr = new Uint8Array([1, 2, 3, 4]);
const copy = arr.slice(1, 3);  // 新しいメモリ領域
```

**外部メモリ（External Memory）**

Buffer のバッキングストアは V8 ヒープの外に存在する（`external` メモリ）。

```javascript
const used = process.memoryUsage();
// used.arrayBuffers → ArrayBuffer/SharedArrayBuffer/Buffer のバッキングストアが使用するメモリ量
// used.external に含まれる
```

**エンコーディング**

Buffer はバイト列を保持する。そのバイト列に「意味」を与えるのがエンコーディング。

```javascript
const buf = Buffer.from('hello', 'utf-8');
console.log(buf.toString('hex'));    // 68656c6c6f
console.log(buf.toString('base64')); // aGVsbG8=
console.log(buf.toString('utf-8'));  // hello
```

### ③ なぜ効果的か

Buffer が V8 ヒープ外のメモリを使うため、`process.memoryUsage().heapUsed` だけ見ても実際のメモリ使用量は分からない。`arrayBuffers` の値も監視する必要がある。

---

## 5. Buffer アロケーション — プールと安全なメモリ管理

**URL:** https://www.thenodebook.com/buffers/allocation-patterns

### ① 目的・問題意識

`Buffer.alloc` / `Buffer.allocUnsafe` / `Buffer.from` の違いと、安全なアロケーションの原則を理解する。

### ② 技術詳細

```javascript
// Buffer.alloc(size) → ゼロ初期化（安全だが遅い）
const safe = Buffer.alloc(1024);

// Buffer.allocUnsafe(size) → 未初期化（速いが前の内容が残る危険あり）
const fast = Buffer.allocUnsafe(1024);
// 必ず即座に全バイトを上書きすること！

// Buffer.from(string) → プールを使う場合あり（コピー保証なし）
const buf = Buffer.from(someString, 'utf-8');
// 内部実装に依存してビューになる可能性 → 元データ変更で「不変」なはずの Buffer が化ける

// Buffer.concat(list) → 毎回コピーを生成
// ❌ ストリーミング処理での繰り返し concat は非効率
let result = Buffer.alloc(0);
for (const chunk of chunks) {
  result = Buffer.concat([result, chunk]); // N回のアロケーション + N-1回のコピー
}

// ✅ 一括 concat（チャンクを貯めてから一度だけ）
const finalBuffer = Buffer.concat(chunks);
```

**原則**

1. デフォルトは `alloc()` を使う
2. プロファイラで必要が判明した場合のみ `allocUnsafe()` を使い、その場合は即座に全領域を上書きすること
3. `from()` には何を渡すかを常に意識する

### ③ なぜ効果的か

`allocUnsafe` は OS が以前使用したメモリをゼロ化せずに返すため、機密データが漏洩するリスクがある。安全性とパフォーマンスのトレードオフを意識的に選ぶことが重要。

---

## 6. Buffer の断片化・保持ビュー・外部メモリ問題

**URL:** https://www.thenodebook.com/buffers/fragmentation-and-challenges

### ① 目的・問題意識

小さい Buffer スライスが大きなバッファ全体をメモリに残し続ける「保持ビュー」問題と、それが引き起こすメモリリーク・OOM を理解する。

### ② 技術詳細

**保持ビュー（Retained Views）問題**

```javascript
// 数MBのバッファを受信
const largeBuffer = Buffer.allocUnsafe(10 * 1024 * 1024); // 10MB

// 先頭16バイトだけのスライスを保存
const smallSlice = largeBuffer.slice(0, 16);

// largeBuffer への参照はなくなるが...
// smallSlice が生きている限り、10MB のバッキングストア全体が GC されない!
```

**メモリ診断の方法**

```javascript
// arrayBuffers が上がり続ける → 保持ビューのライフタイムバグ
// RSS が上がり続けるが arrayBuffers は安定 → アロケータ or ネイティブメモリの問題
const usage = process.memoryUsage();
console.log(`heapUsed: ${usage.heapUsed / 1024 / 1024} MB`);
console.log(`arrayBuffers: ${usage.arrayBuffers / 1024 / 1024} MB`);
console.log(`external: ${usage.external / 1024 / 1024} MB`);
console.log(`rss: ${usage.rss / 1024 / 1024} MB`);
```

**断片化（Fragmentation）との違い**

- **断片化**: アロケータが再利用できるメモリが非連続に分散している（アロケータの問題）
- **保持ビュー**: JS オブジェクトが大きなバッキングストアを参照し続けている（参照ライフタイムの問題）

症状が似ているが原因が異なるため、診断に注意が必要。

### ③ なぜ効果的か

コンテナや Node プロセスの OOM は「JavaScript ヒープが小さく見えるのにメモリ不足でクラッシュ」という形で現れる。`arrayBuffers` と RSS の両方を監視し、ヒープスナップショットで保持ビューの根本参照を追跡することが重要。

---

## 7. ストリームの基礎 — チャンク、バッファ、バックプレッシャー

**URL:** https://www.thenodebook.com/streams/foundation-of-streams

### ① 目的・問題意識

ストリームがメモリ効率よく大きなデータを扱える仕組みと、バックプレッシャーがパイプライン全体に伝播する仕組みを理解する。

### ② 技術詳細

```
Readable → Transform → Transform → Writable
  |                                    |
  ← ← ← バックプレッシャーの伝播 ← ← ←
```

各ステージがチャンクを逐次処理。Writable の内部バッファが満杯になると：
1. Writable が Transform に「停止」を伝える
2. Transform が自分の消費元 Readable を一時停止させる
3. Readable がデータソースの読み取りを止める

全メモリ使用量が `highWaterMark` のサイズに抑えられ、ファイルサイズに関係なく動作する。

### ③ なぜ効果的か

`fs.readFileSync` や `Buffer.concat` でファイル全体を読み込む代わりにストリームを使うことで、メモリ使用量を定数に保てる。特に大ファイル変換・圧縮・暗号化パイプラインで効果的。

---

## 8. Readable ストリーム — モード、バッファ、バックプレッシャー

**URL:** https://www.thenodebook.com/streams/readable-streams

### ① 目的・問題意識

`flowing` / `paused` の2モードの内部動作と、`highWaterMark` による自動バックプレッシャーを理解する。

### ② 技術詳細

**モード遷移**

```javascript
const readable = fs.createReadStream('file.txt');
// readable.readableFlowing === null（初期状態）

readable.on('data', (chunk) => {  // → flowing モード（readableFlowing = true）
  console.log(chunk);
});

readable.pause();  // → paused モード（readableFlowing = false）
readable.resume(); // → flowing モードに戻る
```

**`_read()` とバックプレッシャー**

```javascript
const { Readable } = require('stream');

class MyReadable extends Readable {
  constructor() {
    super({ highWaterMark: 16 * 1024 }); // 16KB バッファ上限
    this.data = generateData();
    this.index = 0;
  }

  _read(size) {  // size はヒント（無視してもよい）
    const chunk = this.data[this.index++];
    if (chunk === undefined) {
      this.push(null); // ストリーム終端
      return;
    }
    const canContinue = this.push(chunk);
    // push() が false を返したらバッファが highWaterMark に達した
    // _read() は再度呼ばれるまで何もしない（バックプレッシャー適用）
    if (!canContinue) return;
  }
}
```

**内部バッファの動作**

- push() → バッファに積む（`state.length` 増加）
- read()/data イベント → バッファから取り出す（`state.length` 減少）
- `state.length >= state.highWaterMark` → push() が `false` を返す → バックプレッシャー発動

### ③ なぜ効果的か

ソースが消費者より速い場合でも、`highWaterMark` でバッファ上限を制御し、メモリが際限なく増えないようにできる。

---

## 9. Transform ストリーム — `_transform` と `_flush`

**URL:** https://www.thenodebook.com/streams/transform-streams

### ① 目的・問題意識

入力チャンクを受け取って変換し出力するパイプラインステージを実装する方法と、ストリーム終端での未処理データの確定方法を理解する。

### ② 技術詳細

```javascript
const { Transform } = require('stream');

// NDJSON → オブジェクト変換の例
class NDJSONParser extends Transform {
  constructor() {
    super({ objectMode: true });
    this._buffer = '';
  }

  _transform(chunk, encoding, callback) {
    this._buffer += chunk.toString();
    const lines = this._buffer.split('\n');
    this._buffer = lines.pop(); // 最後の不完全な行を保持

    for (const line of lines) {
      if (line.trim()) {
        try {
          this.push(JSON.parse(line));
        } catch (e) {
          return callback(e); // エラー時はコールバックにエラーを渡す
        }
      }
    }
    callback(); // 処理完了を通知
  }

  _flush(callback) {
    // ストリーム終端で残りデータを確定
    if (this._buffer.trim()) {
      try {
        this.push(JSON.parse(this._buffer));
      } catch (e) {
        return callback(e);
      }
    }
    callback();
  }
}

// パイプラインでの使用
fs.createReadStream('data.ndjson')
  .pipe(new NDJSONParser())
  .pipe(new FilterTransform())
  .pipe(new WritableTarget());
```

**バッチ処理 Transform**（バッファリングが必要なケース）

```javascript
class BatchTransform extends Transform {
  constructor(batchSize = 100) {
    super({ objectMode: true });
    this._batch = [];
    this._batchSize = batchSize;
  }

  _transform(item, encoding, callback) {
    this._batch.push(item);
    if (this._batch.length >= this._batchSize) {
      this.push(this._batch.splice(0, this._batchSize));
    }
    callback();
  }

  _flush(callback) {
    if (this._batch.length > 0) {
      this.push(this._batch); // 残ったアイテムをフラッシュ
    }
    callback();
  }
}
```

**`_flush` が必要なケース**

- チャンク境界をまたぐパーサー（CSV, NDJSON など）
- バッファリングして一定量まで溜めてから流す場合
- 状態を持つ Transform 全般

**`_flush` が不要なケース**

- 各チャンクを独立して変換する場合（gzip 圧縮・暗号化・フォーマット変換など）

### ③ なぜ効果的か

`_flush` を実装し忘れると、ストリーム終端でバッファに残っているデータが静かに失われる。パーサーや状態を持つ Transform では必須。

---

## 10. ES モジュール — import/export と静的リンク

**URL:** https://www.thenodebook.com/modules/esm-import-export

### ① 目的・問題意識

ESM の静的解析・ライブバインディング・循環依存の挙動を理解し、CJS との本質的な違いを把握する。

### ② 技術詳細

**静的解析（Static Analysis）**

```javascript
// ESM: import/export は静的宣言
import { readFile } from 'node:fs/promises';
export const VERSION = '1.0.0';

// ❌ ESM では動的インポートパスは使えない（静的でない）
// import something from variable;  // 構文エラー

// ✅ 動的インポートが必要な場合は import() 式を使う
const module = await import(path);
```

ESM では依存グラフ全体がコード実行前に構築される。これにより：
- コード実行前に「存在しないエクスポートの import」がエラーとして検出される
- バンドラーがツリーシェイキングで未使用コードを除去できる

**ライブバインディング（Live Bindings）**

```javascript
// counter.mjs
export let count = 0;
export function increment() { count++; }

// app.mjs
import { count, increment } from './counter.mjs';
console.log(count); // 0
increment();
console.log(count); // 1 ← ライブバインディングで最新値が見える
```

CJS では `require()` が実行時スナップショットを取るため、この動作は実現できない。

**循環依存と ESM**

```
A.mjs → import B.mjs
B.mjs → import A.mjs（循環）
```

ESM ではバインディングスロットがコード実行前に作成・相互リンクされる。一方が先に評価されるとき、もう一方のバインディングは `undefined` だが、評価が進むにつれてライブバインディング経由で最新値が見える。

### ③ なぜ効果的か

ライブバインディングにより「インポート後に export 先が変わった」状態を自動追跡でき、シングルトンモジュールの状態共有が安全にできる。また静的解析によりリンク時エラーが早期検出される。

---

## 11. CJS/ESM 相互運用とデュアルパッケージ

**URL:** https://www.thenodebook.com/modules/cjs-esm-interop

### ① 目的・問題意識

CJS と ESM が混在する npm パッケージで発生する「デュアルパッケージ問題」（同一パッケージの二重ロード）を理解し、解決策を知る。

### ② 技術詳細

**デュアルパッケージ問題**

同じパッケージが CJS と ESM の両エントリポイントを持つ場合、`require()` と `import` の両方から使われると2つの別々のモジュールインスタンスが作られる。シングルトン・コネクションプール・設定キャッシュなどは二重化される。

**package.json の `exports` フィールド（Node v24+ 推奨）**

```json
{
  "name": "my-lib",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.js",
      "default": "./dist/cjs/index.js"
    }
  },
  "main": "./dist/cjs/index.js",
  "types": "./dist/index.d.ts",
  "sideEffects": false
}
```

**Node v24+ での `require(esm)` 対応**

フラグなしで同期 ESM を `require()` できるようになった。これにより ESM のみのパッケージを CJS コードから使えるケースが増えた。

```javascript
// Node 24 以降（フラグ不要）
const { something } = require('./esm-module.mjs');
```

**モジュール解決アルゴリズム**

```javascript
// bare specifier の解決順序
require('express')
// 1. node_modules/express/package.json の "exports" フィールド確認（最優先）
// 2. なければ "main" フィールド確認
// 3. ./node_modules → ../node_modules → ../../node_modules ... とルートまで遡る
```

### ③ なぜ効果的か

`exports` フィールドを使うことで、パッケージの公開 API を明示的に制御でき、内部実装の詳細を隠蔽できる。また Node 24+ の `require(esm)` 対応により、デュアルビルドが不要なケースも増えている。

---

## 12. Promise とミクロタスクスケジューリング

**URL:** https://www.thenodebook.com/async-patterns/promises-microtasks

### ① 目的・問題意識

Promise の .then() コールバックがなぜ「次のイベントループ」ではなく「今のマクロタスク直後」に実行されるのかを、V8 の `PromiseReactionJob` と Node の `process.nextTick` の違いとともに理解する。

### ② 技術詳細

**Promise の状態マシン**

```
pending → fulfilled（.resolve(value) 呼び出し時）
pending → rejected（.reject(reason) 呼び出し時）
```

一度決定された状態は変わらない（不変の遷移）。

**PromiseReactionJob**

```javascript
Promise.resolve(42)
  .then(v => v * 2)    // PromiseReactionJob がミクロタスクキューに登録
  .then(v => v + 1)    // チェーン = 10個の .then() = 10個の Promise + 10個のマイクロタスクジョブ
  .then(console.log);
```

**3つのキューの優先順位**

```
process.nextTick キュー （最高優先度：Node 独自）
       ↓（全ドレイン後）
Promise ミクロタスクキュー（V8 管理 PromiseReactionJob）
       ↓（全ドレイン後）
マクロタスク（timers/poll/check フェーズのコールバック）
```

```javascript
setTimeout(() => console.log('setTimeout'), 0);
setImmediate(() => console.log('setImmediate'));
Promise.resolve().then(() => console.log('Promise.then'));
queueMicrotask(() => console.log('queueMicrotask'));
process.nextTick(() => console.log('nextTick'));

// nextTick → Promise.then → queueMicrotask → setImmediate/setTimeout
```

**queueMicrotask**

```javascript
// Promise を作らずにミクロタスクをスケジュールする標準 API（Node v11+）
queueMicrotask(() => {
  // Promise.then() と同じキューで実行される
});
```

**パフォーマンス注意点**

```javascript
// ❌ ホットループで大量の Promise を作る
async function processItems(items) {
  for (const item of items) {
    await Promise.resolve(item); // 毎回 Promise オブジェクトを生成
  }
}

// ✅ 内部はコールバックで、外側だけ Promise でラップ
function processItems(items, callback) {
  // 内部処理（高速）
  callback(null, result);
}
const processItemsAsync = util.promisify(processItems);
```

### ③ なぜ効果的か

ミクロタスクキューはマクロタスク（I/O コールバック等）より常に優先して全ドレインされる。これにより Promise チェーンが「現在の操作に論理的に属する」コードを即座に実行できる一方、過度な .then() チェーンはメモリ（Promise オブジェクトの山）と GC 圧力を生む。

---

## 13. Node.js プロセスライフサイクル — 起動・シグナル・シャットダウン・終了コード

**URL:** https://www.thenodebook.com/node-arch/node-process-lifecycle

### ① 目的・問題意識

プロセスの起動から終了までのライフサイクルを理解し、Kubernetes などのオーケストレーターが期待するグレースフルシャットダウンを正しく実装する。

### ② 技術詳細

**シグナル一覧**

| シグナル | 用途 |
|---|---|
| `SIGINT` | Ctrl+C（インタラクティブ終了要求） |
| `SIGTERM` | オーケストレーター（Kubernetes/systemd）からの正常終了要求 |
| `SIGHUP` | デーモンの設定再読み込みトリガー |
| `SIGKILL` | キャッチ不可能（OS が即座に強制終了） |

**グレースフルシャットダウンの実装パターン**

```javascript
let isShuttingDown = false;

async function gracefulShutdown(signal) {
  if (isShuttingDown) return; // 冪等性の保証
  isShuttingDown = true;
  process.exitCode = 0;

  console.log(`Received ${signal}, starting graceful shutdown...`);

  // 強制終了タイムアウト（Kubernetes の grace period に合わせる）
  const forceExit = setTimeout(() => {
    console.error('Graceful shutdown timed out, forcing exit');
    process.exit(1);
  }, 10_000);
  forceExit.unref(); // タイムアウトだけのためにプロセスを保持しない

  try {
    await server.close();          // 新規接続の受付を停止
    await db.disconnect();         // DB 接続をクローズ
    await messageQueue.drain();    // インフライトのメッセージを処理完了
    clearTimeout(forceExit);
    console.log('Graceful shutdown complete');
  } catch (err) {
    console.error('Error during shutdown:', err);
    process.exitCode = 1;
  }
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// 未処理の例外・Rejection を補足
process.on('uncaughtException', (err) => {
  console.error('Uncaught exception:', err);
  gracefulShutdown('uncaughtException');
});

process.on('unhandledRejection', (reason) => {
  console.error('Unhandled rejection:', reason);
  gracefulShutdown('unhandledRejection');
});
```

**ESM でのトップレベル await を使った起動**

```javascript
// ESM ではトップレベル await が使える（CJS では使えない）
import { connectToDatabase } from './database.js';
const db = await connectToDatabase();
startServer(db);
```

**CJS での起動パターン**

```javascript
async function main() {
  const db = await connectToDatabase();
  startServer(db);
}

main().catch((error) => {
  console.error('Startup failed:', error);
  process.exitCode = 1;
});
```

**意味のある終了コード**

```javascript
// DB 接続失敗を専用コードで区別
process.exitCode = 2; // DB 接続失敗
process.exitCode = 3; // 設定読み込み失敗
// 0 で終了したら「正常完了」とスーパーバイザーが判断することを忘れずに
```

### ③ なぜ効果的か

SIGTERM ハンドラがないと Kubernetes がグレース期間後に SIGKILL を送り、インフライトリクエストが途中で切れる。適切な終了コードがないとスーパーバイザーが失敗を正常終了と誤解する。1つのシャットダウンマネージャーに集約し、モジュールが個別に `process.on('SIGTERM')` を登録するパターンは避けること。

---

## 14. 標準 I/O — ファイルディスクリプタ、TTY、パイプ、バックプレッシャー

**URL:** https://www.thenodebook.com/process-os/standard-io

### ① 目的・問題意識

`stdout` の振る舞いが TTY（ターミナル）とパイプで異なる理由と、パイプ時のバックプレッシャーを理解する。

### ② 技術詳細

**標準ディスクリプタ**

```
fd 0 → process.stdin
fd 1 → process.stdout
fd 2 → process.stderr
```

**TTY vs パイプの違い**

| 接続先 | libuv ハンドル | 書き込み動作 |
|---|---|---|
| TTY（ターミナル） | `uv_tty_t` | Unix では同期ブロッキング `write(2)` |
| パイプ / ファイル / ソケット | `uv_pipe_t` | イベントループに参加、非同期書き込みキューを持つ |

```javascript
// 実行コンテキストの検出
if (process.stdout.isTTY) {
  // ターミナル → カラー出力などが使える
  process.stdout.write('\x1b[32mGreen text\x1b[0m\n');
} else {
  // パイプ → プレーンテキスト
  process.stdout.write('Plain text\n');
}
```

**パイプ時のバックプレッシャー**

```bash
# stdout が gzip に接続 → 圧縮速度に合わせてバックプレッシャーが発生
node app.js | gzip > output.gz

# head が 10 行で停止 → パイプが閉じられ EPIPE エラーが発生する可能性
node app.js | head -n 10
```

```javascript
// EPIPE エラーのハンドリング
process.stdout.on('error', (err) => {
  if (err.code === 'EPIPE') process.exit(0); // 正常な終了として扱う
});
```

### ③ なぜ効果的か

CI/CD やデプロイパイプライン、ログ収集システムでは stdout がパイプに接続される。TTY 前提の実装をするとパイプ時にバックプレッシャーや EPIPE エラーで予期せず失敗する。

---

## 15. ネットワーク基礎 — TCP/UDP/HTTP と net モジュール

**URL:** https://www.thenodebook.com/browse/networking

### ① 目的・問題意識

Node.js が OS のソケットをどのように抽象化しているか、TCP 接続から HTTP リクエストに至る流れを libuv レベルから理解する。

### ② 技術詳細

**TCP サーバー内部**

```javascript
const net = require('net');

const server = net.createServer((socket) => {
  // socket = libuv の uv_tcp_t ハンドルをラップした Duplex ストリーム
  socket.on('data', (data) => {
    socket.write(data); // エコー
  });

  socket.on('end', () => {
    socket.destroy();
  });
});

server.listen(3000, '127.0.0.1', () => {
  console.log('Server listening on port 3000');
});
```

**接続確立フロー**

```
hostname 解決（DNS lookup）
      ↓
アドレス選択（IPv4/IPv6）
      ↓
ルーティング
      ↓
TCP connect（3ウェイハンドシェイク）
      ↓
accept キュー
      ↓
libuv 準備完了イベント
      ↓
JavaScript コールバック（connection イベント）
```

**ソケットオプション**

```javascript
const socket = net.connect({ port: 3000 });
socket.setKeepAlive(true, 60000);  // TCP keepalive（60秒間隔）
socket.setNoDelay(true);           // Nagle アルゴリズム無効化（低遅延重視）
```

**UDP（Datagram）**

```javascript
const dgram = require('dgram');
const server = dgram.createSocket('udp4');

server.on('message', (msg, rinfo) => {
  console.log(`Received from ${rinfo.address}:${rinfo.port}: ${msg}`);
  server.send('ACK', rinfo.port, rinfo.address);
});

server.bind(41234);
```

UDP の特徴：接続確立不要、フレーミングは自前、パケット順序・到達保証なし。

### ③ なぜ効果的か

net/dgram モジュールの内部が libuv ハンドルと直結していることを理解すると、HTTP ライブラリや WebSocket の動作を深く理解でき、パフォーマンスチューニングやカスタムプロトコル実装に活用できる。

---

## まとめ

Node Weekly Issue 624 は「NodeBook: An Advanced Guide to Node.js Internals」を特集号として取り上げた。  
V8 の JIT パイプライン・イベントループのフェーズ詳細・Buffer/Stream の内部実装・CJS/ESM の相互運用・プロセスライフサイクルまで、Node.js を本番で使う開発者が知っておくべき低レイヤーの知識を体系的に網羅している。

**関連リンク**
- [Node Weekly Issue 624](https://nodeweekly.com/issues/624)
- [NodeBook (thenodebook.com)](https://www.thenodebook.com/)
- [NodeBook: What is Node.js](https://www.thenodebook.com/node-arch/what-is-nodejs)
- [NodeBook: V8 Engine](https://www.thenodebook.com/node-arch/v8-engine-intro)
- [NodeBook: Event Loop](https://www.thenodebook.com/node-arch/event-loop-intro)
- [NodeBook: Buffers Index](https://www.thenodebook.com/browse/buffers)
- [NodeBook: Streams Index](https://www.thenodebook.com/browse/streams)
- [NodeBook: ESM import/export](https://www.thenodebook.com/modules/esm-import-export)
- [NodeBook: CJS/ESM Interop](https://www.thenodebook.com/modules/cjs-esm-interop)
- [NodeBook: Promises & Microtasks](https://www.thenodebook.com/async-patterns/promises-microtasks)
- [NodeBook: Process Lifecycle](https://www.thenodebook.com/node-arch/node-process-lifecycle)
- [NodeBook: Standard I/O](https://www.thenodebook.com/process-os/standard-io)
- [NodeBook: Networking Index](https://www.thenodebook.com/browse/networking)
