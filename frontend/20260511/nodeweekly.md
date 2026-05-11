# Node Weekly Issue 623 (May 7, 2026) まとめ

> 出典: https://nodeweekly.com/issues/623

---

## 1. Node.js 26.0.0 (Current) リリース

**記事リンク:** https://nodejs.org/en/blog/release/v26.0.0  
**リリース日:** 2026年5月5日

### ① 目的・問題意識

Node.js 26 は「Current」チャンネルの最新メジャーリリース。2026年10月にLTS(長期サポート)へ昇格予定。V8エンジンの更新、Temporal APIの標準化、HTTP クライアントの刷新、および長年の非推奨APIの削除という4つの大きな課題を一括で解決する。

### ② 具体的な技術詳細とコード

#### Temporal API — デフォルトで有効化

Temporal は JavaScript の `Date` オブジェクトの設計上の欠陥（ミュータビリティ、タイムゾーン処理の不備、月が0始まりなど）を解消する次世代日付/時刻 API。Node.js 26 でフラグなしで利用可能になった。

**主要な型:**

| 型 | 用途 |
|---|---|
| `Temporal.PlainDate` | タイムゾーンなしの日付 |
| `Temporal.PlainTime` | タイムゾーンなしの時刻 |
| `Temporal.PlainDateTime` | タイムゾーンなしの日時 |
| `Temporal.ZonedDateTime` | タイムゾーン付きの日時 |
| `Temporal.Instant` | UTC の正確な時刻（ナノ秒精度） |
| `Temporal.Duration` | 時間の長さ |
| `Temporal.PlainYearMonth` | 年月（日なし） |
| `Temporal.PlainMonthDay` | 月日（年なし、誕生日など） |

**コード例:**

```javascript
// PlainDate: タイムゾーンを気にしない日付（誕生日、締め切りなど）
const birthday = Temporal.PlainDate.from('1990-07-15');
console.log(birthday.year);  // 1990
console.log(birthday.month); // 7
console.log(birthday.day);   // 15

// PlainTime: 時刻のみ（営業時間など）
const openTime = Temporal.PlainTime.from('09:00:00');
const closeTime = Temporal.PlainTime.from('18:00:00');

// ZonedDateTime: タイムゾーン付き（国際対応が必要な場合）
const meetingTokyo = Temporal.ZonedDateTime.from('2026-05-10T14:00:00[Asia/Tokyo]');
const meetingNY = meetingTokyo.withTimeZone('America/New_York');
console.log(meetingNY.toString()); // 2026-05-10T01:00:00-04:00[America/New_York]

// Instant: 正確な時刻の記録（ログ、イベント時刻）
const now = Temporal.Now.instant();
console.log(now.epochMilliseconds); // Unix timestamp (ミリ秒)
console.log(now.epochNanoseconds);  // ナノ秒精度（BigInt）

// Duration: 期間の計算
const duration = Temporal.Duration.from({ days: 30, hours: 5, minutes: 30 });
const future = Temporal.Now.plainDateTimeISO().add(duration);
console.log(future.toString());

// 日付の比較（Dateと違いイミュータブル）
const d1 = Temporal.PlainDate.from('2026-01-01');
const d2 = Temporal.PlainDate.from('2026-06-01');
console.log(Temporal.PlainDate.compare(d1, d2)); // -1 (d1 < d2)

// 日付の差分計算
const diff = d1.until(d2);
console.log(diff.toString()); // P5M (5ヶ月)

// 旧来の Date との相互変換
const legacyDate = new Date();
const temporal = Temporal.Instant.fromEpochMilliseconds(legacyDate.getTime());
console.log(temporal.toString());
```

#### V8 14.6 — 新しい JavaScript 機能

**Map/WeakMap Upsert メソッド (TC39 Proposal)**

「キーがなければ挿入して返す」パターンを簡潔に書けるようになった。

```javascript
// 旧来のパターン（冗長）
const cache = new Map();
function getOrCreate(key, factory) {
  if (!cache.has(key)) {
    cache.set(key, factory());
  }
  return cache.get(key);
}

// getOrInsert: 固定値で初期化
const map = new Map();
const val1 = map.getOrInsert('key', 'default');
console.log(val1);         // 'default'
console.log(map.get('key')); // 'default'

const val2 = map.getOrInsert('key', 'other'); // 既存値を返す
console.log(val2);         // 'default'

// getOrInsertComputed: ファクトリ関数で初期化（コスト高い生成に有用）
const expensiveMap = new Map();
const result = expensiveMap.getOrInsertComputed('user:123', (key) => {
  console.log(`Computing value for ${key}`);
  return { id: key, createdAt: Date.now() };
});

// WeakMap でも同様
const weakMap = new WeakMap();
const obj = {};
const weakVal = weakMap.getOrInsert(obj, []);
weakVal.push('item');
```

**Iterator.concat() (Iterator Sequencing)**

複数のイテレータを結合してひとつのシーケンスにする。

```javascript
// 複数のデータソースを統合
const numbers = [1, 2, 3].values();
const letters = ['a', 'b', 'c'].values();
const symbols = ['!', '@', '#'].values();

const combined = Iterator.concat(numbers, letters, symbols);
console.log([...combined]); // [1, 2, 3, 'a', 'b', 'c', '!', '@', '#']

// ジェネレータとの組み合わせ
function* evens() { yield 2; yield 4; yield 6; }
function* odds() { yield 1; yield 3; yield 5; }

const allNumbers = Iterator.concat(odds(), evens());
for (const n of allNumbers) {
  console.log(n); // 1, 3, 5, 2, 4, 6
}
```

#### Undici 8.0.2 — fetch の基盤更新

Node.js のグローバル `fetch()` を支える HTTP クライアント。v8 では接続管理、HTTP/2 サポート、ストリーミングの信頼性が向上。

```javascript
// fetch は Undici 8 で動作（コードは変わらないが信頼性が向上）
const response = await fetch('https://api.example.com/data', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ query: 'hello' }),
});
const data = await response.json();
```

#### 破壊的変更（SEMVER-MAJOR）

```javascript
// ❌ 削除: http.Server.prototype.writeHeader()
server.writeHeader(200, { 'Content-Type': 'text/plain' });

// ✅ 代替: writeHead() を使う
server.writeHead(200, { 'Content-Type': 'text/plain' });

// ❌ 削除: レガシー内部ストリームモジュール
const readable = require('_stream_readable'); // Error
const writable = require('_stream_writable'); // Error

// ✅ 代替: 標準 stream モジュール
const { Readable, Writable } = require('stream');
```

**ビルド要件の変更:**
- GCC 13.2 以上が必要（以前は 12.x）
- Python 3.9 のサポート終了（Python 3.10 以上が必要）
- Windows SDK: バージョン 11 が必要
- `NODE_MODULE_VERSION` が 147 に更新

### ③ なぜ効果的か

- **Temporal API**: `Date` の可変性・タイムゾーンバグ・DST (夏時間) の落とし穴を根本から解消する。Node.js 26 でフラグなし利用可能になったことで、本番コードへの移行が現実的に。
- **Map.getOrInsert()**: キャッシュやメモ化パターンで毎回 `has()` + `set()` + `get()` の3操作が1操作になり、コードの可読性と安全性が向上。
- **Undici 8**: HTTP クライアントの信頼性向上により、`fetch()` を使う全てのコードに透過的に恩恵がある。

---

## 2. Node.js 26.1.0 (Current) リリース — 実験的 FFI モジュール

**記事リンク:** https://nodejs.org/en/blog/release/v26.1.0  
**リリース日:** 2026年5月7日（26.0.0 と同日）

### ① 目的・問題意識

Node.js から C ライブラリ（DLL/shared library）を呼び出すには従来 C++ アドオンの記述が必要で、ビルドチェーンが煩雑だった。`node:ffi` モジュールはこの問題を解消し、Deno や Bun が先行実装していたネイティブライブラリ直接呼び出しを Node.js でも実現する。

### ② 具体的な技術詳細とコード

#### `node:ffi` — Foreign Function Interface

```bash
# ビルド時（ソースからのビルドが必要）
./configure.py --with-ffi
# Windows の場合
./vcbuild.bat ffi

# 実行時（両フラグが必須）
node --experimental-ffi --allow-ffi app.js
```

```javascript
const { dlopen, UnsafeCallback, UnsafePointer, UnsafePointerView } = require('node:ffi');

// Windows の user32.dll を読み込み、EnumWindows を呼び出す例
const lib = dlopen('user32', {
  EnumWindows: {
    result: 'i32',
    parameters: ['pointer', 'u64']
  }
});

// JavaScript から C に渡すコールバックを作成
const callback = new UnsafeCallback({
  result: 'i32',
  parameters: ['pointer', 'u64']
}, (hWnd, lParam) => {
  console.log('Window handle:', hWnd, 'lParam:', lParam);
  return 1n; // 列挙を継続
});

// ネイティブ関数を呼び出す
lib.symbols.EnumWindows(callback.pointer, 3n);

// DynamicLibrary は Symbol.dispose をサポート（using 構文で自動クリーンアップ）
using const safeLib = dlopen('mylib', { myFunc: { result: 'void', parameters: [] } });
safeLib.symbols.myFunc();
// スコープ終了時に自動アンロード
```

**エクスポートされる主要な API:**

| API | 用途 |
|---|---|
| `dlopen(name, symbols)` | 動的ライブラリのロードとシンボル定義 |
| `UnsafeCallback` | JS 関数を C コールバックポインタとして渡す |
| `UnsafePointer` | 生メモリアドレスの管理 |
| `UnsafePointerView` | メモリの読み書き |

**プラットフォームサポート:**

| OS | x64 | ARM32 | ARM64 |
|---|---|---|---|
| Windows | ✓ | ✓ | ✓ |
| Linux | ✓ | ✓ | ✓ |
| macOS | ✓ | — | ✓ |

> ⚠️ **安全性警告**: 無効なポインタ、誤った型定義、解放済みメモリへのアクセスはプロセスのクラッシュやメモリ破壊を引き起こす。

#### `crypto.randomUUIDv7()` — UUID v7 生成

```javascript
const crypto = require('crypto'); // または node:crypto

// UUIDv7 = 時刻順ソート可能な UUID（Postgres や Bun と互換）
const id = crypto.randomUUIDv7();
console.log(id);
// 例: "01970e86-1234-7abc-8def-0123456789ab"
// 先頭48ビット = Unix タイムスタンプ（ミリ秒）→ 自然順ソートが可能

// UUIDv4 との比較（ランダム、ソート不可）
const v4 = crypto.randomUUID();
console.log(v4);
// 例: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
```

#### Buffer 改善 — 検索範囲の指定

```javascript
const buf = Buffer.from('Hello, World! Hello!');

// end パラメータが追加され、検索範囲を限定できるようになった
const idx1 = buf.indexOf('Hello', 0);    // 0
const idx2 = buf.indexOf('Hello', 0, 13); // 0 (end=13 で最初の "Hello" のみ)
const idx3 = buf.indexOf('Hello', 0, 5);  // -1 (範囲外)

// lastIndexOf も同様
const lastIdx = buf.lastIndexOf('Hello', buf.length, 7); // 14
```

#### HTTP 改善 — `req.signal`

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  // IncomingMessage に AbortSignal が追加された
  req.signal.addEventListener('abort', () => {
    console.log('クライアントが接続を切断しました');
    // 長い処理をキャンセルできる
  });

  // signal を使って非同期処理をキャンセル
  fetch('https://api.example.com/slow', { signal: req.signal })
    .then(data => res.end(JSON.stringify(data)))
    .catch(err => {
      if (err.name === 'AbortError') {
        res.destroy();
      }
    });
});
```

#### テストランナー — 実行順序のランダム化

```javascript
// 実行順序をランダムにしてテストの依存関係のバグを検出
// node --test --experimental-test-randomize-order test.js

import { test } from 'node:test';
import assert from 'node:assert';

test('test A', () => {
  assert.strictEqual(1 + 1, 2);
});

test('test B', () => {
  assert.strictEqual(2 * 3, 6);
});

// テストが実行順序に依存している場合、ランダム化でバグが露出する
```

#### `util.styleText()` — 16進カラーコード対応

```javascript
const { styleText } = require('util');

// 既存: ANSI 色名
console.log(styleText('red', 'エラーメッセージ'));
console.log(styleText(['bold', 'green'], '成功'));

// 新機能: 16進カラーコードに対応
console.log(styleText('#FF5733', 'オレンジ色のテキスト'));
console.log(styleText('#00FF00', '緑色のテキスト'));
```

#### 依存関係の更新

| パッケージ | 旧バージョン | 新バージョン |
|---|---|---|
| V8 | 14.6.202.33 | 14.6.202.34 |
| OpenSSL | - | 3.5.6 |
| npm | - | 11.13.0 |
| undici | 8.0.2 | 8.2.0 |
| SQLite | - | 3.53.0 |
| nghttp2 | - | 1.69.0 |

### ③ なぜ効果的か

- **node:ffi**: C++ アドオンのビルドチェーンなしにネイティブライブラリを呼べるため、音声処理・画像処理・暗号化ライブラリなど高性能ネイティブコードを JavaScript から薄いラッパーで利用可能になる。
- **randomUUIDv7()**: 時刻順ソート可能な UUID はデータベースのインデックス効率（B-tree の書き込みホットスポット軽減）に優れ、PostgreSQL などとの親和性が高い。
- **req.signal**: リクエストキャンセル時に下流の非同期処理（DB クエリ、外部 API コール）も中断できるため、サーバーリソースの無駄遣いを防げる。

---

## 3. Node.js 組み込み SQLite (`node:sqlite`) の進化

**記事リンク:** https://nodejs.org/api/sqlite.html

### ① 目的・問題意識

Node.js に `node:sqlite` が組み込まれたことで、外部の `better-sqlite3` や `sqlite3` パッケージなしに SQLite データベースを操作できる。Node.js 26.x では `SQLTagStore`（プリペアドステートメントキャッシュ）の追加と Percentile 拡張の有効化により、より実用的になった。

### ② 具体的な技術詳細とコード

#### 基本的な使い方

```javascript
import { DatabaseSync } from 'node:sqlite';

// インメモリデータベース
const db = new DatabaseSync(':memory:');

// テーブル作成と挿入
db.exec(`
  CREATE TABLE users (
    id   INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    age  INTEGER
  ) STRICT
`);

const insert = db.prepare('INSERT INTO users (name, age) VALUES (?, ?)');
insert.run('Alice', 30);
insert.run('Bob', 25);

// 全件取得
const all = db.prepare('SELECT * FROM users ORDER BY age').all();
console.log(all);
// [ { id: 2, name: 'Bob', age: 25 }, { id: 1, name: 'Alice', age: 30 } ]

// 1件取得
const user = db.prepare('SELECT * FROM users WHERE name = ?').get('Alice');
console.log(user); // { id: 1, name: 'Alice', age: 30 }

// イテレータ（大量データを省メモリで処理）
for (const row of db.prepare('SELECT * FROM users').iterate()) {
  console.log(row);
}
```

#### `SQLTagStore` — テンプレートリテラルによるキャッシュ付きクエリ

```javascript
import { DatabaseSync } from 'node:sqlite';

const db = new DatabaseSync(':memory:');
db.exec('CREATE TABLE products (id INTEGER PRIMARY KEY, name TEXT, price REAL)');

// SQLTagStore: プリペアドステートメントを LRU キャッシュ
const sql = db.createTagStore(1000); // 最大 1000 件キャッシュ

// テンプレートリテラルで自動的にパラメータ化（SQL インジェクション防止）
const productName = 'Widget';
const price = 9.99;
sql.run`INSERT INTO products (name, price) VALUES (${productName}, ${price})`;
sql.run`INSERT INTO products (name, price) VALUES (${'Gadget'}, ${19.99})`;

// 変数をそのまま渡しても安全（パラメータバインディング）
const search = 'Widget';
const found = sql.get`SELECT * FROM products WHERE name = ${search}`;
console.log(found); // { id: 1, name: 'Widget', price: 9.99 }

// 全件取得
const allProducts = sql.all`SELECT * FROM products ORDER BY price`;
console.log(allProducts);

// キャッシュ情報
console.log(sql.size);     // 現在のキャッシュ数
console.log(sql.capacity); // 最大容量

// イテレータ
for (const product of sql.iterate`SELECT * FROM products WHERE price < ${15}`) {
  console.log(product);
}
```

#### ユーザー定義集計関数 + Percentile 拡張

```javascript
import { DatabaseSync } from 'node:sqlite';

const db = new DatabaseSync(':memory:');
db.exec(`
  CREATE TABLE scores (student TEXT, score REAL);
  INSERT INTO scores VALUES ('Alice', 85), ('Bob', 92), ('Carol', 78),
                            ('Dave', 95), ('Eve', 88), ('Frank', 71);
`);

// カスタム集計関数の例
db.aggregate('sumint', {
  start: 0,
  step: (acc, value) => acc + value,
});
const total = db.prepare('SELECT sumint(score) as total FROM scores').get();
console.log(total); // { total: 509 }

// Percentile 拡張（Node.js 26 で有効化）
// median(), percentile(), percentile_cont(), percentile_disc() が使える
const median = db.prepare('SELECT median(score) as median FROM scores').get();
console.log(median); // { median: 86.5 }

const p90 = db.prepare("SELECT percentile(score, 90) as p90 FROM scores").get();
console.log(p90); // { p90: 93.8 } (90パーセンタイル)
```

#### バックアップとシリアライズ

```javascript
import { backup, DatabaseSync } from 'node:sqlite';

// オンラインバックアップ（書き込み中でも安全）
const db = new DatabaseSync('production.db');
const totalPages = await backup(db, 'backup.db', {
  rate: 100, // 1回の呼び出しでコピーするページ数
  progress: ({ totalPages, remainingPages }) => {
    const pct = Math.round(((totalPages - remainingPages) / totalPages) * 100);
    console.log(`バックアップ進捗: ${pct}%`);
  },
});

// インメモリへシリアライズ（テスト・複製に有用）
const original = new DatabaseSync('source.db');
const buffer = original.serialize(); // Uint8Array
original.close();

const clone = new DatabaseSync(':memory:');
clone.deserialize(buffer);
```

#### セッション（変更追跡・レプリケーション）

```javascript
import { DatabaseSync } from 'node:sqlite';

const db = new DatabaseSync(':memory:');
db.exec('CREATE TABLE events (id INTEGER PRIMARY KEY, type TEXT, ts INTEGER)');

// 変更を記録するセッションを開始
const session = db.createSession();

const ins = db.prepare('INSERT INTO events (type, ts) VALUES (?, ?)');
ins.run('login', Date.now());
ins.run('purchase', Date.now());

// 変更セット（changeset）を取得 → 別のDBに適用可能
const changeset = session.changeset(); // Uint8Array
session.close();

// 別のデータベースに変更を適用（レプリケーション）
const replica = new DatabaseSync(':memory:');
replica.exec('CREATE TABLE events (id INTEGER PRIMARY KEY, type TEXT, ts INTEGER)');
replica.applyChangeset(changeset, {
  onConflict: () => 'replace', // 競合時の解決策
});
```

#### `using` 構文によるリソース管理

```javascript
import { DatabaseSync } from 'node:sqlite';

// using 構文（Explicit Resource Management）でスコープ終了時に自動クローズ
{
  using const db = new DatabaseSync(':memory:');
  db.exec('CREATE TABLE t (x INTEGER)');
  db.prepare('INSERT INTO t VALUES (?)').run(42);
  const row = db.prepare('SELECT * FROM t').get();
  console.log(row); // { x: 42 }
} // ここで db.close() が自動呼び出し
```

### ③ なぜ効果的か

- **依存なし**: `better-sqlite3` などの npm パッケージ不要で、デプロイサイズの削減とセキュリティ脆弱性のリスク低減になる。
- **SQLTagStore**: テンプレートリテラルでパラメータが自動バインドされるため SQL インジェクションが構造的に防げる上、キャッシュでパフォーマンスも改善する。
- **Percentile 拡張**: ログ解析・モニタリングなどで p50/p90/p99 レイテンシを SQLite だけで算出できる。

---

## 4. Map/WeakMap の Upsert パターン詳解

**記事リンク:** https://github.com/nodejs/node/releases/tag/v26.0.0

### ① 目的・問題意識

`Map` でキャッシュやメモ化を実装するとき、「なければセット、あれば既存値を返す」パターンは非常によく使われるが、従来は複数のメソッド呼び出しが必要だった。V8 14.6 の Upsert Proposal でこれを1メソッドで完結できるようになった。

### ② 具体的なコードと技術詳細

```javascript
// === 旧来のパターン ===
const cache = new Map();

// パターン1: has + set + get（3回のルックアップ）
function getOrCreate_old(key, factory) {
  if (!cache.has(key)) {
    cache.set(key, factory(key));
  }
  return cache.get(key);
}

// パターン2: get + undefined チェック（falsy値に対して不正確）
function getOrCreate_old2(key, factory) {
  let val = cache.get(key);
  if (val === undefined) {
    val = factory(key);
    cache.set(key, val);
  }
  return val;
}

// === 新しいパターン ===

// getOrInsert: 固定値で初期化
const registry = new Map();
const entry1 = registry.getOrInsert('config', { debug: false });
// キーがなければ { debug: false } を挿入して返す
// キーがあれば既存値を返す（引数は評価されるが使われない）

// getOrInsertComputed: ファクトリ関数（遅延評価）
const expensiveCache = new Map();
const result = expensiveCache.getOrInsertComputed('user:1', (key) => {
  // キーが存在しない場合のみ呼ばれる（高コスト処理を避けられる）
  return fetchUserFromDB(key);
});

// WeakMap でも同じAPIが使える（GCと連動するキャッシュに有用）
const weakCache = new WeakMap();
const obj = {};
const weakEntry = weakCache.getOrInsert(obj, new Set());
weakEntry.add('value1');

// 実用例: グループ化（reduce の代替）
const data = [
  { category: 'fruit', name: 'apple' },
  { category: 'veggie', name: 'carrot' },
  { category: 'fruit', name: 'banana' },
];

const grouped = new Map();
for (const item of data) {
  grouped.getOrInsertComputed(item.category, () => []).push(item.name);
}
console.log(grouped.get('fruit'));  // ['apple', 'banana']
console.log(grouped.get('veggie')); // ['carrot']
```

### ③ なぜ効果的か

`getOrInsert` は内部で1回のハッシュルックアップで完結するため、`has()` + `set()` + `get()` の3回より効率的。特に `getOrInsertComputed` はキーが存在する場合にファクトリ関数を呼ばないため、重い初期化コストを避けられる。コードの意図も明確になり、バグの混入リスクも下がる。

---

## 5. `node:ffi` — ネイティブライブラリ直接呼び出し詳解

**記事リンク:** https://github.com/nodejs/node/pull/57761

### ① 目的・問題意識

Node.js から C のネイティブライブラリ（.dll / .so / .dylib）を呼び出すには、従来 Node-API (N-API) による C++ アドオンの作成、ビルドツール (`node-gyp`) のセットアップ、プラットフォームごとのコンパイルが必要だった。Deno や Bun が先に搭載していた FFI を Node.js も採用し、この煩雑さを解消する。

### ② 具体的なコードと技術詳細

```javascript
// node --experimental-ffi --allow-ffi app.js
const { dlopen, UnsafeCallback, UnsafePointer, UnsafePointerView } = require('node:ffi');

// === 例1: Linux の libc から strlen を呼び出す ===
const libc = dlopen('libc', {
  strlen: {
    result: 'usize',
    parameters: ['pointer']
  }
});

// === 例2: Windows user32.dll の EnumWindows ===
const user32 = dlopen('user32', {
  EnumWindows: {
    result: 'i32',
    parameters: ['pointer', 'u64']
  },
  GetWindowTextA: {
    result: 'i32',
    parameters: ['pointer', 'pointer', 'i32']
  }
});

const windowCallback = new UnsafeCallback({
  result: 'i32',
  parameters: ['pointer', 'u64']
}, (hWnd, lParam) => {
  console.log('ウィンドウハンドル:', hWnd);
  return 1n; // 1 = 列挙継続, 0 = 停止
});

user32.symbols.EnumWindows(windowCallback.pointer, 0n);

// コールバックを使い終わったら解放（GC対策）
windowCallback.close();

// === 型定義リスト ===
// 'void', 'bool'
// 'u8', 'i8', 'u16', 'i16', 'u32', 'i32', 'u64', 'i64'
// 'f32', 'f64'
// 'usize', 'isize'
// 'pointer', 'buffer', 'function'

// === Symbol.dispose によるリソース管理 ===
{
  using const lib = dlopen('mylib', {
    compute: { result: 'f64', parameters: ['f64', 'f64'] }
  });
  const result = lib.symbols.compute(3.14, 2.71);
  console.log(result);
} // 自動アンロード
```

**サポートされる型の対応表:**

| FFI 型 | JavaScript 型 |
|---|---|
| `i32`, `u32` | `number` |
| `i64`, `u64` | `BigInt` |
| `f32`, `f64` | `number` |
| `pointer` | `UnsafePointer` / `null` |
| `buffer` | `ArrayBuffer` / `TypedArray` |
| `bool` | `boolean` |

### ③ なぜ効果的か

- C++ アドオンのビルドチェーンなしに、音声ライブラリ（PortAudio）、画像処理（OpenCV）、ML ランタイム（ONNX）などを Node.js から直接呼び出せる。
- libffi を使った実装で、ABI レベルの型変換を自動処理。
- `Symbol.dispose` 対応により、`using` 構文でライブラリのライフタイムを安全に管理できる。

---

## 技術サマリー

| トピック | バージョン | 重要度 |
|---|---|---|
| Temporal API デフォルト有効化 | Node.js 26.0.0 | ★★★ |
| V8 14.6 (getOrInsert / Iterator.concat) | Node.js 26.0.0 | ★★☆ |
| Undici 8.0.2 → 8.2.0 | Node.js 26.0.0 / 26.1.0 | ★★☆ |
| 実験的 `node:ffi` モジュール | Node.js 26.1.0 | ★★★ |
| `crypto.randomUUIDv7()` | Node.js 26.1.0 | ★★☆ |
| SQLite Percentile 拡張 | Node.js 26.1.0 | ★★☆ |
| SQLTagStore (テンプレートリテラルキャッシュ) | Node.js 26.x | ★★☆ |
| `req.signal` on IncomingMessage | Node.js 26.1.0 | ★★☆ |
| テストランナー 実行順ランダム化 | Node.js 26.1.0 | ★☆☆ |

---

Sources:
- [Node Weekly Issue 623: May 7, 2026](https://nodeweekly.com/issues/623)
- [Node.js 26.0.0 (Current)](https://nodejs.org/en/blog/release/v26.0.0)
- [Node.js 26.1.0 (Current)](https://nodejs.org/en/blog/release/v26.1.0)
- [Node.js v26.0.0 Release on GitHub](https://github.com/nodejs/node/releases/tag/v26.0.0)
- [Node.js v26.1.0 Release on GitHub](https://github.com/nodejs/node/releases/tag/v26.1.0)
- [node:ffi Initial Implementation PR #57761](https://github.com/nodejs/node/pull/57761)
- [Node.js Built-in SQLite Documentation](https://nodejs.org/api/sqlite.html)
- [Node.js 26 ships with Temporal API - Help Net Security](https://www.helpnetsecurity.com/2026/05/07/node-js-26-released/)
- [Exploring Temporal API - Better Stack](https://betterstack.com/community/guides/scaling-nodejs/temporal-explained/)
- [Node.js v26 Is Here: What Actually Changed - NodeSource](https://nodesource.com/blog/nodejs-v26-is-here)
