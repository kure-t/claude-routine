# Node Weekly Issue 620 まとめ (April 16, 2026)

> 出典: [Node Weekly Issue 620](https://nodeweekly.com/issues/620)

---

## 1. Node が Temporal をデフォルト有効化へ向けて動き出す

**原文リンク:** [Let's enable Temporal by default (GitHub Issue #57127)](https://github.com/nodejs/node/issues/57127) / [PR #61806](https://github.com/nodejs/node/pull/61806)

### ① 目的・問題意識

JavaScript の `Date` オブジェクトは長年にわたり多くの問題を抱えてきた。

- 月が 0 始まり（1月 = `0`）で直感に反する
- タイムゾーンの処理が不透明・バグを生みやすい
- ミリ秒精度しかなく、ナノ秒が扱えない
- `Date` オブジェクトはミュータブル（破壊的変更が可能）
- 夏時間（DST）の扱いが不正確

`Temporal` API はこれらを根本から解決する新しい標準 API であり、TC39 の Stage 4（ES2026 正式採択）に 2026年3月11日に到達した。V8 14.4 でデフォルト有効化され、Node.js においても PR #61806 が 2026年4月13日にマージされ、将来の Node.js 26 でデフォルト有効化される道筋が整った。

### ② 具体的な技術詳細

#### PR #61806 の変更内容

`configure.py` に 2 つの新フラグが追加された：

```bash
# Temporal サポートを明示的に無効化（Rust 不要）
./configure --v8-disable-temporal-support

# Temporal サポートを明示的に有効化（Rust が必要、なければビルドエラー）
./configure --v8-enable-temporal-support
```

**自動検出ロジック：** どちらのフラグも指定しない場合、ビルドシステムが `cargo` と `rustc` の存在を自動検出する。両方見つかれば Temporal が有効になり、見つからなければ警告を出して無効化する（ビルドは継続）。

**なぜ Rust が必要か：** V8 の Temporal 実装は内部で Rust 製のコンポーネント（Unicode カレンダー処理など）を利用しているため。

#### Temporal API の主要な型

```javascript
// ---- Temporal.Now ----
// 現在の日付（ISO 8601、タイムゾーン指定可能）
const today = Temporal.Now.plainDateISO();
console.log(today.toString()); // "2026-04-16"

// 現在の日時（タイムゾーン付き）
const now = Temporal.Now.zonedDateTimeISO('America/New_York');

// ---- Temporal.PlainDate ----
// タイムゾーンなし・時刻なし の日付
const date = Temporal.PlainDate.from('2026-04-16');
console.log(date.year);  // 2026
console.log(date.month); // 4 (1-indexed!)
console.log(date.day);   // 16

// 日付の加算（イミュータブル）
const nextWeek = date.add({ days: 7 });
console.log(nextWeek.toString()); // "2026-04-23"

// ---- Temporal.PlainTime ----
const time = Temporal.PlainTime.from('09:30:00');
console.log(time.hour);   // 9
console.log(time.minute); // 30

// ---- Temporal.PlainDateTime ----
const dt = Temporal.PlainDateTime.from({
  year: 2026, month: 4, day: 16,
  hour: 9, minute: 30,
});

// ---- Temporal.ZonedDateTime ----
// タイムゾーン付き日時（最も安全・推奨）
const zdt = Temporal.ZonedDateTime.from(
  '2026-04-16T09:30:00[America/New_York]'
);
console.log(zdt.timeZoneId); // "America/New_York"

// タイムゾーン変換
const tokyoTime = zdt.withTimeZone('Asia/Tokyo');
console.log(tokyoTime.toString());
// "2026-04-16T22:30:00+09:00[Asia/Tokyo]"

// ---- Temporal.Instant ----
// UTC タイムスタンプ（ナノ秒精度）
const instant = Temporal.Instant.from('2026-04-16T09:30:00Z');
console.log(instant.epochMilliseconds); // Unix ミリ秒
console.log(instant.epochNanoseconds);  // BigInt でナノ秒精度

// ---- Temporal.Duration ----
const duration = Temporal.Duration.from({ years: 1, months: 2, days: 3 });
console.log(duration.toString()); // "P1Y2M3D"

// ---- 期間計算 ----
const start = Temporal.PlainDate.from('2026-01-01');
const end   = Temporal.PlainDate.from('2026-04-16');
const diff  = start.until(end, { largestUnit: 'days' });
console.log(diff.days); // 105

// since() も使える
const diff2 = end.since(start, { largestUnit: 'months' });
console.log(diff2.months); // 3
```

#### Moment.js との対比

```javascript
// ---- Moment.js（旧来） ----
const m = moment();
const mNext = m.add(7, 'days'); // mutable! m 自体が変わる

// ---- Temporal（新しい） ----
const t = Temporal.Now.plainDateISO();
const tNext = t.add({ days: 7 }); // イミュータブル。t は変わらない
```

### ③ なぜ効果的か

- **イミュータブル設計** により副作用バグが根絶される
- **1-indexed の月** で `new Date(2026, 3, 16)` が4月になる罠がなくなる
- **タイムゾーン情報を型で持つ** (`ZonedDateTime`) ことで DST バグが防げる
- **ナノ秒精度** (`Instant.epochNanoseconds` は `BigInt`) で高精度計測が可能
- **外部ライブラリ不要** で Moment.js / date-fns / Luxon からの脱却が可能

---

## 2. Node.js 24.15.0 (LTS) リリース

**原文リンク:** [Node.js 24.15.0 (LTS) Blog Post](https://nodejs.org/en/blog/release/v24.15.0)

リリース日: 2026年4月15日 (コード名: Krypton)

### ① 目的・問題意識

LTS ラインへの機能バックポートと、実験的だった機能の安定化が主目的。特に `require(esm)` と モジュールコンパイルキャッシュの安定化が開発者体験を大幅に改善する。

### ② 主要変更点と技術詳細

#### `require(esm)` の安定化

```javascript
// CommonJS ファイル (.cjs / .js + "type":"commonjs") から
// ESM モジュール (.mjs / .js + "type":"module") を直接 require できる

// main.cjs
const chalk = require('chalk'); // chalk は ESM-only パッケージ
console.log(chalk.green('Hello!'));
```

**重要な制約:** `require()` は同期処理のため、Top-Level Await (TLA) を持つ ESM モジュールは `require()` できない。TLA を使う場合は `import()` (動的インポート) を使うこと。

#### モジュールコンパイルキャッシュの安定化

```javascript
// programmatic API
import { enableCompileCache } from 'node:module';
enableCompileCache(); // 呼び出し後にロードされるモジュールがキャッシュされる

// または環境変数で指定
// NODE_COMPILE_CACHE=/tmp/node-cache node app.js
```

コンパイルキャッシュにより、V8 のバイトコードをディスクに保存し、次回起動時に再コンパイルをスキップする。例として `typescript.js` のロードが約 130ms → 約 80ms に短縮された。

#### `--max-heap-size` CLI オプション

```bash
# ヒープサイズを 512MB に制限
node --max-heap-size=512 app.js

# 2GB に設定
node --max-heap-size=2048 server.js
```

以前は `--max-old-space-size` で近い設定ができたが、より直感的な名前の公式フラグが追加された。

#### `fs.stat()` の `throwIfNoEntry` オプション

```javascript
const fs = require('fs');

// ファイルが存在しない場合に undefined を返す（エラーを投げない）
const stats = fs.statSync('maybe-exists.txt', { throwIfNoEntry: false });
if (stats) {
  console.log(stats.size);
} else {
  console.log('ファイルが存在しません');
}

// Promises 版
const stats2 = await fs.promises.stat('maybe-exists.txt', { throwIfNoEntry: false });
```

#### Socket TOS (Type of Service) 制御

```javascript
const net = require('net');

const socket = net.createConnection({ port: 8080 }, () => {
  // QoS のための TOS ビットを設定
  socket.setTOS(0x10); // Minimize-Delay (低遅延を優先)
  console.log('TOS:', socket.getTOS()); // 16
});
```

#### HTTP/2 の HTTP/1 フォールバック設定

```javascript
const http2 = require('http2');

const server = http2.createSecureServer({
  // HTTP/2 非対応クライアントへの HTTP/1 フォールバック設定
  http1Options: {
    keepAlive: true,
    keepAliveTimeout: 5000,
  },
  allowHTTP1: true,
}, handler);
```

#### `Duplex.toWeb()` の `readableType` リネーム

```javascript
const { Duplex } = require('stream');

// 旧: type オプション（非推奨）
// duplex.toWeb({ type: 'bytes' });

// 新: readableType オプション
const { readable, writable } = duplex.toWeb({ readableType: 'bytes' });
```

#### SQLite の `limits` プロパティ追加

```javascript
const { DatabaseSync } = require('node:sqlite');
const db = new DatabaseSync(':memory:');

// データベースの制限値を取得
const limits = db.limits;
console.log(limits);
// {
//   SQLITE_LIMIT_LENGTH: 1000000000,
//   SQLITE_LIMIT_SQL_LENGTH: 1000000000,
//   ...
// }
```

SQLite モジュールはこのリリースでリリース候補 (RC) ステータスに昇格。

#### テストランナーの改善

```javascript
import { test } from 'node:test';

test('parallel test', async (t) => {
  // Worker ID にアクセス可能
  const { workerId } = await import('node:test');
  console.log(`Running in worker ${workerId}`);
});

// SIGINT 受信時に中断中のテスト名を表示するように改善
```

#### その他の依存関係更新

| パッケージ | バージョン |
|-----------|-----------|
| npm | 11.12.1 |
| SQLite | 3.52.0 |
| ada (URL パーサー) | 3.4.4 |
| NSS (ルート証明書) | 3.121 |

#### 非推奨 (DEP0206)

```javascript
// 非推奨: node:crypto での CryptoKey 直接使用
const { createSecretKey } = require('crypto');

// 推奨: KeyObject を使う
const key = createSecretKey(Buffer.from('secret', 'utf8'));
// key は KeyObject インスタンス
```

### ③ なぜ効果的か

- **`require(esm)` の安定化** により ESM-only npm パッケージへの段階的移行が可能になり、コードベース全体を一度に書き換えなくてよい
- **コンパイルキャッシュ** により Lambda 等のコールドスタート時間を削減できる
- **`--max-heap-size`** でメモリ制約の厳しい環境でのチューニングが容易になる
- **`throwIfNoEntry`** でファイル存在確認のボイラープレートが削減される

---

## 3. Moment.js から Temporal API への移行ガイド

**原文リンク:** [Moving From Moment.js To The JS Temporal API — Smashing Magazine](https://www.smashingmagazine.com/2026/03/moving-from-moment-to-temporal-api/)

著者: Joe Attardi

### ① 目的・問題意識

Moment.js は長年 JavaScript の日付処理の標準として使われてきたが、メンテナンスモードに入り、バンドルサイズが大きく（min+gzip で約 18KB）、ミュータブルな API 設計がバグを生みやすい。Temporal API が Stage 4 になったタイミングで移行する実践的なレシピを提供する。

### ② 具体的なコードと移行パターン

#### 現在日時の取得

```javascript
// Moment.js
const now = moment();
console.log(now.format()); // "2026-04-16T09:30:00-05:00"

// Temporal
const now = Temporal.Now.zonedDateTimeISO('America/New_York');
console.log(now.toString()); // "2026-04-16T09:30:00-05:00[America/New_York]"
```

#### 日付の解析

```javascript
// Moment.js
const date = moment('2026-02-21');

// Temporal
const date = Temporal.PlainDate.from('2026-02-21');

// RFC 9557 形式（タイムゾーン付き）
const instant = Temporal.Instant.from('2026-02-21T09:00:00-05:00[America/New_York]');
```

#### フォーマット（表示）

```javascript
// Moment.js
const formatted = moment().format('M/D/YYYY'); // "4/16/2026"

// Temporal (Intl.DateTimeFormat を利用)
const date = Temporal.Now.plainDateISO();
const formatOptions = { month: 'numeric', day: 'numeric', year: 'numeric' };
console.log(date.toLocaleString('en-US', formatOptions)); // "4/16/2026"
console.log(date.toLocaleString('ja-JP', formatOptions)); // "2026/4/16"
```

#### 日付の加算・減算

```javascript
// Moment.js（ミュータブル！）
const date = moment('2026-04-16');
date.add(7, 'days'); // date 自体が変わる
console.log(date.format('YYYY-MM-DD')); // "2026-04-23"

// Temporal（イミュータブル）
const date = Temporal.PlainDate.from('2026-04-16');
const nextWeek = date.add({ days: 7 }); // date は変わらない
console.log(nextWeek.toString()); // "2026-04-23"
console.log(date.toString());     // "2026-04-16" (変更なし)
```

#### 日付の比較

```javascript
// Moment.js
const a = moment('2026-04-16');
const b = moment('2026-05-01');
console.log(a.isBefore(b)); // true
console.log(a.diff(b, 'days')); // -15

// Temporal
const a = Temporal.PlainDate.from('2026-04-16');
const b = Temporal.PlainDate.from('2026-05-01');
console.log(Temporal.PlainDate.compare(a, b)); // -1 (a < b)
console.log(a.until(b).days); // 15
```

#### タイムゾーン変換

```javascript
// Moment.js（moment-timezone が必要）
const nyTime = moment.tz('2026-04-16T09:30:00', 'America/New_York');
const tokyoTime = nyTime.clone().tz('Asia/Tokyo');

// Temporal（ライブラリ不要）
const nyTime = Temporal.ZonedDateTime.from(
  '2026-04-16T09:30:00[America/New_York]'
);
const tokyoTime = nyTime.withTimeZone('Asia/Tokyo');
console.log(tokyoTime.toString()); // "2026-04-16T22:30:00+09:00[Asia/Tokyo]"
```

### ③ なぜ効果的か

- Temporal はネイティブ API のため **バンドルサイズゼロ**
- **イミュータブル** なのでサプライズな副作用がない
- **1-indexed の月** (`month: 4` が4月) で直感的
- `Intl.DateTimeFormat` と直接統合されロケール対応が簡単
- タイムゾーンライブラリ（moment-timezone 等）が**不要**

---

## 4. Node.js 20 が 2026年4月30日に End of Life を迎える

**関連リンク:** [Node.js EOL Dates](https://endoflife.date/nodejs) / [Migration Playbook](https://dev.to/matheus_releaserun/nodejs-20-end-of-life-migration-playbook-for-april-30-2026-2onh)

### ① 目的・問題意識

Node.js 20 (LTS) のサポートが 2026年4月30日に終了する。EOL 後はセキュリティパッチや CVE 修正が提供されなくなるため、本番環境での継続使用は重大なリスクになる。

### ② 技術詳細：移行手順

#### 移行先の選択

| バージョン | 状態 | サポート終了 |
|-----------|------|------------|
| Node.js 20 | **EOL (2026/4/30)** | ~~2026-04-30~~ |
| Node.js 22 | Active LTS | 2027-04-30 |
| Node.js 24 | LTS (最新) | 2028-04-30 |

#### 移行チェックリスト

```bash
# 1. 現在の Node.js バージョン確認
node -v  # v20.x.x なら移行が必要

# 2. nvm でバージョン切り替え
nvm install 22
nvm use 22

# 3. ネイティブアドオンの再ビルド（必要な場合）
npm rebuild

# 4. 特定パッケージの再インストール（ABI が変わるもの）
npm install sharp@latest   # 例: sharp はネイティブモジュール
npm install bcrypt@latest

# 5. テスト実行
npm test
```

#### 主な破壊的変更（Node.js 20 → 22 移行時）

```javascript
// import assertions 構文の変更
// 旧 (Node.js 20 まで動作)
import data from './data.json' assert { type: 'json' };

// 新 (Node.js 22+)
import data from './data.json' with { type: 'json' };
```

```bash
# AWS Lambda の場合
# Phase 1: 2026/4/30 - セキュリティパッチ停止
# Phase 2: 2026/8/31 - 新規関数の nodejs20.x ランタイム作成不可
# Phase 3: 2026/9/30 - 既存関数の更新不可（完全廃止）
```

### ③ なぜ効果的か

- Node.js 22 に移行することで **2027年4月まで** Active LTS のサポートを受けられる
- Node.js 22 には require(esm)、Native TypeScript サポート（フラグ付き）など 20 にない機能がある
- **ほとんどの場合、変更は設定ファイル 1 行の変更** で済む（`.nvmrc`、`Dockerfile`、CI 設定）

---

## まとめ

| トピック | 重要度 | アクション |
|---------|--------|-----------|
| Temporal API デフォルト化 (Node.js 26 予定) | ★★★ | Moment.js からの移行計画を立てる |
| Node.js 24.15.0 LTS リリース | ★★★ | require(esm) とコンパイルキャッシュを活用する |
| Moment.js → Temporal 移行ガイド | ★★☆ | 新規コードは Temporal を使い始める |
| Node.js 20 EOL (2026/4/30) | ★★★ | **今すぐ** Node.js 22 または 24 に移行する |
