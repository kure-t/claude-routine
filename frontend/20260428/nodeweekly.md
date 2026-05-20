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

## 5. x-win: Node からウィンドウ情報を取得する

**原文リンク:** [x-win on GitHub](https://github.com/miniben-90/x-win)

### ① 目的・問題意識

デスクトップアプリや開発ツールを Node.js で作る際、現在アクティブなウィンドウや開いているウィンドウの情報（タイトル、位置、サイズ、プロセス情報）をプログラムから取得したいケースがある。従来は OS ごとに異なるネイティブ API を叩く必要があり、クロスプラットフォーム対応が困難だった。`x-win` はこれを Rust + napi-rs で抽象化し、Windows / macOS / Linux で統一した API を提供する。

### ② 具体的な技術詳細

#### インストール

```bash
npm i @miniben90/x-win
```

#### 基本的な使い方

```javascript
import { activeWindow, openWindows } from '@miniben90/x-win';

// 現在アクティブなウィンドウの情報を取得
const win = activeWindow();
console.log(win);
// {
//   id: 12345,
//   title: 'Visual Studio Code',
//   processPath: '/usr/bin/code',
//   processName: 'code',
//   pid: 9876,
//   os: 'linux',
//   position: { x: 0, y: 0, width: 1920, height: 1080 },
//   memoryUsage: 512000,
// }

// 全ウィンドウの一覧取得
const windows = openWindows();
windows.forEach(w => console.log(w.title));
```

#### 非同期版

```javascript
import { activeWindowAsync, openWindowsAsync } from '@miniben90/x-win';

const win = await activeWindowAsync();
const windows = await openWindowsAsync();
```

#### アクティブウィンドウの変化を監視（100ms ポーリング）

```javascript
import { subscribeActiveWindow } from '@miniben90/x-win';

const unsubscribe = subscribeActiveWindow((win) => {
  console.log('アクティブウィンドウが変わりました:', win.title);
});

// 監視停止
setTimeout(() => unsubscribe(), 10000);
```

#### アイコン取得

```javascript
import { getIcon } from '@miniben90/x-win';

const win = activeWindow();
const iconBase64 = getIcon(win.id); // base64 PNG
```

#### ブラウザの URL 取得（macOS / Windows のみ）

```javascript
const win = activeWindow();
if (win.url) {
  console.log('現在開いているURL:', win.url);
}
```

#### サポートプラットフォーム

| OS | 条件 |
|----|------|
| Windows 10+ | フル対応、ブラウザ URL 取得可能 |
| macOS 10.6+ | Catalina 以降は画面収録権限が必要 |
| Linux (X11) | `libxcb-dev` 等が必要 |
| Linux (Wayland/GNOME) | `installExtension()` で拡張機能をインストール |

### ③ なぜ効果的か

- Rust + napi-rs によりネイティブ性能でウィンドウ情報を取得
- 単一の npm パッケージで Windows / macOS / Linux に対応
- `subscribeActiveWindow` により Electron 不要で時間計測ツールや集中管理ツールが作れる

---

## 6. DocMD: Markdown から本番対応ドキュメントサイトを生成

**原文リンク:** [docmd.io](https://docmd.io/) / [GitHub](https://github.com/docmd-io/docmd)

### ① 目的・問題意識

Docusaurus や VitePress は高機能だが、設定が複雑でバンドルサイズが大きくなりがち。`DocMD` は「Markdown を書くだけ」で SEO 対応・多言語・バージョン管理・オフライン検索付きの静的ドキュメントサイトを生成する。クライアント JS は約 18KB のみで Lighthouse スコア 100 を達成する。

### ② 具体的な技術詳細

#### インストールとクイックスタート

```bash
npm install -g @docmd/core

# 開発サーバー起動（http://localhost:3000）
docmd dev

# 本番ビルド
docmd build

# 既存ドキュメントを移行（Docusaurus / VitePress / MkDocs 対応）
docmd migrate
```

#### 最小構成（設定ファイル不要）

```
my-docs/
├── docs/
│   ├── index.md
│   └── guide.md
└── package.json
```

`npx @docmd/core dev` だけで動く。

#### 設定ファイル（オプション）

```javascript
// docmd.config.js
const { defineConfig } = require('@docmd/core');

module.exports = defineConfig({
  title: 'My Project Docs',
  url: 'https://docs.myproject.com',
  versions: {
    current: 'v2',
    all: [
      { id: 'v2', dir: 'docs' },
      { id: 'v1', dir: 'docs-v1' },
    ],
  },
  i18n: {
    defaultLocale: 'en',
    locales: ['en', 'ja', 'de', 'fr'],
  },
});
```

#### デプロイ設定の自動生成

```bash
# Docker 用 Dockerfile + nginx.conf を生成
docmd deploy --docker
# ✓ Generated Dockerfile
# ✓ Generated nginx.conf
# → docker build -t docs .

# Nginx 単体
docmd deploy --nginx

# Caddy
docmd deploy --caddy
```

#### プログラマティック API

```javascript
const { build } = require('@docmd/core');
await build('./docmd.config.js', { isDev: false });
```

#### 主な機能

| 機能 | 詳細 |
|------|------|
| 検索 | オフラインのファジー全文検索（ロケール別インデックス） |
| i18n | ロケール優先 URL、翻訳済み UI 文字列 |
| バージョン切替 | docs-v1 / docs-v2 など複数ディレクトリ対応 |
| AI コンテキスト | `llms.txt` を自動生成 |
| プラグイン | Mermaid 図、PWA、LaTeX 数式、アナリティクス |

### ③ なぜ効果的か

- React/Vue 不要で **クライアント JS が約 18KB** のみ、高速表示
- `docmd deploy --docker` 一発で本番デプロイ設定が揃う
- Docusaurus 等からの **自動マイグレーション** コマンドで移行コストが低い

---

## 7. TinyTTS: 3.4MB の軽量英語テキスト読み上げ

**原文リンク:** [GitHub - tronghieuit/tiny-tts](https://github.com/tronghieuit/tiny-tts)

### ① 目的・問題意識

テキスト読み上げ (TTS) には通常クラウド API（Google Cloud TTS、AWS Polly 等）か大型モデルが必要で、オフライン・エッジ環境での利用が難しかった。`TinyTTS` は ONNX Runtime を使った約 3.4MB（FP16）の超軽量モデルで、CPU のみで **約 53倍リアルタイム** の速度で英語音声を合成する。

### ② 具体的な技術詳細

#### 仕様

| 項目 | 値 |
|------|-----|
| モデルサイズ | ~3.4MB（FP16 ONNX） |
| パラメータ数 | ~1.6M |
| サンプリングレート | 44.1 kHz |
| 処理速度 | ~53倍リアルタイム（CPU） |
| ライセンス | Apache 2.0 |

#### Node.js インストールと使用例

```bash
npm install tiny-tts
```

```javascript
const TinyTTS = require('tiny-tts');

const tts = new TinyTTS();

await tts.speak('Hello, this is Node Weekly.', {
  outputPath: 'output.wav',
  speed: 1.0,
  speaker: 'MALE',
});
```

#### Python でも同じモデルを使用可能

```python
from tiny_tts import TinyTTS

tts = TinyTTS()
tts.speak("Hello, this is a test.", output_path="hello.wav", speed=1.5)
```

#### CLI

```bash
tiny-tts --text "Sample text" --output output.wav --speed 1.0
```

#### アーキテクチャ

- Grapheme-to-Phoneme (G2P) 変換 → CMU 発音辞書（123,463 エントリ）
- Viterbi アライメント
- エンドツーエンド音声合成（ボコーダー不要）
- ONNX Runtime で実行（CPU のみで動作）

### ③ なぜ効果的か

- **3.4MB** という極小サイズで IoT・エッジデバイスにデプロイ可能
- **クラウド API 不要**のためオフライン環境やコスト削減に有効
- Node.js と Python で同一モデルを共有できる

---

## 8. Marked.js 18.0: 高速 Markdown パーサーが TypeScript 6 対応

**原文リンク:** [marked on npm](https://www.npmjs.com/package/marked) / [GitHub Releases](https://github.com/markedjs/marked/releases)

### ① 目的・問題意識

`marked` はブラウザ・サーバー両対応の高速 Markdown パーサーとして広く使われており（12,000+ 依存パッケージ）、v18.0.0 では TypeScript 6 への対応と、パース精度に関わるバグ修正が行われた。

### ② 具体的な技術詳細

#### インストール

```bash
npm i marked
```

#### 基本的な使い方

```javascript
import { marked } from 'marked';

const html = marked('# Hello World\n\nThis is **marked** v18.');
console.log(html);
// <h1>Hello World</h1>
// <p>This is <strong>marked</strong> v18.</p>
```

#### v18.0.0 の破壊的変更

**1. ブロックトークン末尾の空行をトリム**

以前はトークンに末尾の空行が含まれていたが、v18.0.0 からトリムされる。カスタムレンダラーやプラグインで末尾の空行に依存していた場合は修正が必要。

**2. TypeScript を v5.9.3 → v6.0.2 にアップグレード**

```bash
npx tsc --version  # Version 6.0.2
```

#### v18.0.2 のバグ修正

インデント付きコードブロックと空行の組み合わせで無限ループが発生するバグが修正された。

```markdown
    indented code block

    (blank line)

    more indented code
```

#### カスタムレンダラーの例

```javascript
import { marked, Renderer } from 'marked';

const renderer = new Renderer();

renderer.heading = ({ text, depth }) => {
  const id = text.toLowerCase().replace(/\s+/g, '-');
  return `<h${depth} id="${id}">${text}</h${depth}>\n`;
};

marked.use({ renderer });
console.log(marked('# Hello World'));
// <h1 id="hello-world">Hello World</h1>
```

### ③ なぜ効果的か

- **TypeScript 6 対応**によりモダンな TypeScript プロジェクトで型エラーが出なくなる
- 末尾空行のトリムにより **パース結果の一貫性** が向上し、プラグイン開発が予測しやすくなる
- v18.0.2 の無限ループ修正により **本番環境での安全性** が向上

---

## 9. Node.js リリーススケジュールの進化

**原文リンク:** [Evolving the Node.js Release Schedule](https://nodejs.org/en/blog/announcements/evolving-the-nodejs-release-schedule)

### ① 目的・問題意識

現行の Node.js は年 2 回のメジャーリリース（奇数番 = Current、偶数番 = LTS）を行っているが、奇数番リリースの採用率が低く、同時に複数ラインをメンテするボランティアの負担が大きかった。Node.js 27 (2027年) から年 1 回のリリースモデルに移行する。

### ② 具体的な技術詳細

#### 新しいリリースフロー

```
Alpha (6ヶ月: 10月〜3月)
  ↓  semver-major 変更可能・signed タグ付きリリース
Current (6ヶ月: 4月〜10月)
  ↓  安定化フェーズ
LTS (30ヶ月)
  ↓
EOL
```

#### タイムライン例（Node.js 27）

| マイルストーン | 日付 |
|---------------|------|
| Alpha 開始 | 2026年10月 |
| 27.0.0 リリース | 2027年4月 |
| LTS 昇格 | 2027年10月 |
| End of Life | 2030年4月 |

#### Alpha チャンネルの仕様

```bash
# Alpha はセマンティックバージョンのプレリリース形式
# 例: 27.0.0-alpha.1, 27.0.0-alpha.2, ...

npm install node@27.0.0-alpha.1  # ライブラリ作者向けの早期テスト用
```

- **目的**: 奇数番リリースの代わりに破壊的変更の早期フィードバックを得る
- **対象**: ライブラリ作者、CI パイプライン（本番環境向けではない）
- **品質ゲート**: CITGM（Canary in the Goldmine）テストで品質担保

#### 変わらないこと

- LTS のサポート期間（約 30ヶ月）は維持
- 4月リリース・10月 LTS 昇格のスケジュールは維持
- V8 採用サイクル（Current リリース時点で約 6ヶ月前の V8）

#### 移行スケジュール

| バージョン | 備考 |
|-----------|------|
| Node.js 26 | **現行スケジュール最後のリリース** |
| Node.js 27 | **新スケジュール最初のリリース**（2027年4月） |

### ③ なぜ効果的か

- **奇数番リリース廃止**により新規参入者の混乱がなくなる
- **全リリースが LTS** になることで、どのバージョンにアップグレードしても長期サポートが保証される
- メンテナーの**並行ライン数が減少**しバックポート負担が軽減される

---

## まとめ

| トピック | 重要度 | アクション |
|---------|--------|-----------|
| Temporal API デフォルト化 (Node.js 26 予定) | ★★★ | Moment.js からの移行計画を立てる |
| Node.js 24.15.0 LTS リリース | ★★★ | require(esm) とコンパイルキャッシュを活用する |
| Moment.js → Temporal 移行ガイド | ★★☆ | 新規コードは Temporal を使い始める |
| Node.js 20 EOL (2026/4/30) | ★★★ | **今すぐ** Node.js 22 または 24 に移行する |
| x-win | ★★☆ | デスクトップツール開発に活用する |
| DocMD | ★★☆ | 新規ドキュメントサイトの選択肢として検討する |
| TinyTTS | ★★☆ | オフライン TTS が必要な場面で活用する |
| Marked.js 18.0 | ★★☆ | v18 へアップグレードし無限ループバグを解消する |
| Node.js リリーススケジュール変更 | ★★★ | Node.js 27 以降は全リリースが LTS になると覚えておく |
