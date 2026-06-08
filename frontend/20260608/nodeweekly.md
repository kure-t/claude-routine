# Node Weekly Issue 627 (June 4, 2026) まとめ

> 原文: https://nodeweekly.com/issues/627

---

## 1. replacements.fyi — npmパッケージの代替を見つけるツール

**出典:** https://replacements.fyi/

### ① 目的・問題意識

Node.jsエコシステムには、わずか数行で書けるロジックを担う超軽量npmパッケージが大量に存在する。`is-number`、`chalk`、`axios`などは何百万ものプロジェクトに依存されているが、今のNode.jsはそのほとんどをネイティブAPIで代替できる。不要な依存関係はインストール時間・バンドルサイズ・セキュリティ攻撃面の拡大につながる。`replacements.fyi`は [e18e（ecosystem performance）](https://e18e.dev/) コミュニティが提供する検索ツールで、パッケージ名を入力するだけで「より軽量な代替案」や「ネイティブNode.js APIへの置き換えコード」を提示する。

### ② 具体的なコード・技術詳細

**`is-number` → インラインone-liner**

```js
// ❌ Before
const isNumber = require('is-number');
isNumber(5); // true

// ✅ After (one-liner)
const isNumber = n => typeof n === 'number' && !isNaN(n);
isNumber(5); // true
```

**`axios` → ネイティブ `fetch()`**

```js
// ❌ Before
const axios = require('axios');
const { data } = await axios.get('https://api.example.com/users');

// ✅ After (Node.js 18+標準 fetch)
const res = await fetch('https://api.example.com/users');
const data = await res.json();

// タイムアウトはAbortControllerで
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000);
const res = await fetch('https://api.example.com/users', {
  signal: controller.signal,
});
clearTimeout(timeout);
```

**`chalk` → `util.styleText()`（Node.js 20.12+）**

```js
// ❌ Before
const chalk = require('chalk');
console.log(chalk.red('エラー!'));
console.log(chalk.bold.green('成功!'));

// ✅ After
const { styleText } = require('util');
console.log(styleText('red', 'エラー!'));
console.log(styleText(['bold', 'green'], '成功!'));

// Node.js 24.16+ では16進数カラーも対応
console.log(styleText('#FF5733', 'カスタムカラー'));
```

**`moment` → `Temporal API`（Node.js 26+）**

```js
// ❌ Before
const moment = require('moment');
const date = moment('2026-06-04').add(7, 'days').format('YYYY-MM-DD');

// ✅ After (Temporal API - Node.js 26でデフォルト有効)
const date = Temporal.PlainDate.from('2026-06-04').add({ days: 7 });
console.log(date.toString()); // "2026-06-11"
```

サイトは [module-replacements](https://github.com/e18e/module-replacements) データベースを利用しており、ESLintプラグインやCLIとしても利用可能。

### ③ なぜ効果的か

- 依存関係を削減することで `npm install` の速度向上・バンドルサイズ縮小
- サードパーティパッケージ経由のサプライチェーン攻撃リスクを減らせる（後述のMiasma攻撃参照）
- Node.js標準APIはメンテナンス・セキュリティパッチがランタイムと一体化しており、追加管理不要

---

## 2. Node.js 26.0.0 (Current) リリース

**出典:** https://nodejs.org/en/blog/release/v26.0.0  
**リリース日:** 2026年5月5日 / **リリースマネージャー:** Rafael Gonzaga

### ① 目的・問題意識

Node.js 26は「Current」ステータスで提供される最新安定版（2026年10月にLTS昇格予定）。主な目標は、長年フラグ付きで実験的だったTemporal APIをデフォルト有効化し、JavaScriptの日付処理の課題を解決すること。

### ② 具体的なコード・技術詳細

**Temporal API デフォルト有効化（最大の目玉）**

`Date`オブジェクトに代わる近代的な日時APIが標準で利用可能に。

```js
// タイムゾーン対応の日時
const now = Temporal.Now.zonedDateTimeISO('Asia/Tokyo');
console.log(now.toString());
// "2026-06-04T10:30:00+09:00[Asia/Tokyo]"

// 算術演算
const date = Temporal.PlainDate.from('2026-05-05');
const later = date.add({ months: 1, days: 10 });
console.log(later.toString()); // "2026-06-15"

// 期間の比較
const dt1 = Temporal.PlainDateTime.from('2026-06-04T09:00');
const dt2 = Temporal.PlainDateTime.from('2026-06-04T17:30');
const duration = dt1.until(dt2);
console.log(duration.toString()); // "PT8H30M"
```

**V8 14.6 — Map Upsert API**

```js
// Map.prototype.getOrInsert() — キーが無ければ挿入して値を返す
const cache = new Map();
const value = cache.getOrInsert('key', 'default');
console.log(value);           // 'default'
console.log(cache.get('key')); // 'default'

// getOrInsertComputed() — ファクトリ関数版
const computed = cache.getOrInsertComputed('otherKey', () => expensiveComputation());

// WeakMapでも同様
const wm = new WeakMap();
const obj = {};
wm.getOrInsertComputed(obj, () => ({ created: Date.now() }));
```

**V8 14.6 — Iterator.concat()**

```js
// 複数のイテレーターをシーケンシャルに結合
function* gen1() { yield 1; yield 2; }
function* gen2() { yield 3; yield 4; }

const combined = Iterator.concat(gen1(), gen2(), [5, 6]);
console.log([...combined]); // [1, 2, 3, 4, 5, 6]
```

**主要な削除・非推奨化**

```js
// ❌ 削除: http.Server.prototype.writeHeader()
res.writeHeader(200, { 'Content-Type': 'text/html' }); // 削除

// ✅ 代替: writeHead()
res.writeHead(200, { 'Content-Type': 'text/html' });

// ❌ 削除: レガシー _stream_* モジュール群
const r = require('_stream_readable'); // 削除

// ✅ 代替: 標準 stream モジュール
const { Readable } = require('stream');
```

**ネイティブ拡張の再コンパイル**

NODE_MODULE_VERSIONが147に変わるため、ネイティブアドオンはすべて再コンパイルが必要。

```bash
npm rebuild
```

**ビルド要件の変更**

- GCC 13.2以上が必須
- Python 3.10以上が必須（3.9のサポート終了）

### ③ なぜ効果的か

Temporal APIの標準化により、`moment`・`date-fns`・`dayjs`などの依存が不要になる。V8 14.6のMap Upsert操作はキャッシュ系コードを1行に簡略化し、`Iterator.concat`はジェネレーターを使う処理の構成性を高める。

---

## 3. Node.js 26.1.0 (Current) リリース — 実験的 `node:ffi` モジュール

**出典:** https://nodejs.org/en/blog/release/v26.1.0  
**リリース日:** 2026年5月7日 / **リリースマネージャー:** Antoine du Hamel

### ① 目的・問題意識

JavaScriptからネイティブライブラリ（`.so`/`.dylib`/`.dll`）を呼び出すには従来 N-API でC/C++アドオンを書く必要があった。`node:ffi` は Python の `ctypes` に相当する FFI（Foreign Function Interface）をNode.js本体に組み込み、C拡張を書かずにネイティブコードと連携できるようにする。

### ② 具体的なコード・技術詳細

**有効化フラグ**

```bash
node --experimental-ffi app.js
# Permission Model使用時は --allow-ffi も必要
node --experimental-ffi --allow-ffi app.js
```

**基本的な使い方 — `dlopen`**

```js
import { dlopen, suffix } from 'node:ffi';

// プラットフォーム対応のライブラリパス
const libPath = `./libmath.${suffix}`; // macOS: .dylib, Linux: .so, Win: .dll

const { lib, functions } = dlopen(libPath, {
  add_i32: {
    arguments: ['i32', 'i32'],
    return: 'i32',
  },
  multiply_f64: {
    arguments: ['f64', 'f64'],
    return: 'f64',
  },
});

console.log(functions.add_i32(20, 22));         // 42
console.log(functions.multiply_f64(3.14, 2.0)); // 6.28

lib.close();
```

**`using` 構文による自動クローズ（TC39 Explicit Resource Management）**

```js
import { dlopen } from 'node:ffi';

{
  using handle = dlopen('./mylib.so', {
    process_data: { arguments: ['buffer', 'i32'], return: 'i32' },
  });
  const buf = Buffer.from([1, 2, 3, 4]);
  console.log(handle.functions.process_data(buf, buf.length));
} // ← スコープ終了時に自動的に handle.lib.close() が呼ばれる
```

**DynamicLibrary クラスでの詳細な制御**

```js
import { DynamicLibrary } from 'node:ffi';

const lib = new DynamicLibrary('./libsqlite3.so');

// 個別に関数を解決
const sqlite3_open = lib.getFunction('sqlite3_open', {
  arguments: ['string', 'pointer'],
  return: 'i32',
});

// 複数まとめて解決
const fns = lib.getFunctions({
  strlen:  { arguments: ['string'], return: 'u64' },
  toupper: { arguments: ['i32'],    return: 'i32' },
});

console.log(fns.strlen('hello')); // 5n (u64はbigint)
```

**64ビット整数の取り扱い（bigint）**

```js
// i64/u64 の引数・戻り値は bigint
const { functions } = dlopen('./libmath.so', {
  factorial: { arguments: ['i64'], return: 'i64' },
});
console.log(functions.factorial(20n)); // 2432902008176640000n
```

**コールバック登録（JS→ネイティブ）**

```js
import { DynamicLibrary } from 'node:ffi';
const lib = new DynamicLibrary('./libsort.so');

const comparator = lib.registerCallback(
  { arguments: ['i32', 'i32'], return: 'i32' },
  (a, b) => a - b
);

// ネイティブのqsort等に渡す
const sort = lib.getFunction('sort_with_comparator', {
  arguments: ['buffer', 'i32', 'function'],
  return: 'void',
});

const data = Buffer.from(new Int32Array([3, 1, 4, 1, 5]));
sort(data, 5, comparator);

lib.unregisterCallback(comparator);
lib.close();
```

**メモリアクセスヘルパー**

```js
import { getInt32, setInt32, toBuffer, toString as ffiToString } from 'node:ffi';

// ポインタからプリミティブ値を読み書き
setInt32(ptr, 0, 42);
console.log(getInt32(ptr, 0)); // 42

// ポインタをBufferに変換（コピーあり）
const buf = toBuffer(ptr, 64);

// NUL終端UTF-8文字列を読む
const str = ffiToString(charPtr);
```

**対応する型名一覧**

| 型名 | 説明 |
|------|------|
| `void` | 戻り値なし |
| `i8`〜`i64` | 符号付き整数 |
| `u8`〜`u64` | 符号なし整数 |
| `f32`/`f64` | 浮動小数点 |
| `pointer`/`ptr` | 生ポインタ（bigint） |
| `string`/`str` | NUL終端UTF-8 |
| `buffer` | Buffer/TypedArray |
| `function` | コールバック関数 |

> **安全性の注意:** 無効なポインタ・不正なシグネチャ・解放済みメモリへのアクセスはプロセスのクラッシュやメモリ破損を引き起こす。本番環境での使用は慎重に。

### ③ なぜ効果的か

N-APIアドオンはビルドツールチェーン（node-gyp、Makefile、C++コンパイラ）を要求するため、開発・CI環境の構築コストが高い。`node:ffi`はJSだけでOS提供のシステムライブラリ（`libc`、`libssl`、`libsqlite3`など）に直接アクセスでき、アドオン開発なしにネイティブ性能を活用できる。

---

## 4. `crypto.randomUUIDv7()` の実装

**出典:** https://nodejs.org/en/blog/release/v26.1.0 (Node.js 26.1 / 24.16)

### ① 目的・問題意識

UUID v4はランダムなため、データベースのインデックスに挿入するとページ分断（ページスプリット）が発生しやすい。UUID v7は時刻をプレフィックスに持つため**時系列ソート可能**で、RDBやNoSQLの挿入パフォーマンスが向上する。従来は`uuid`パッケージの`v7()`を使う必要があったが、Node.js本体に組み込まれた。

### ② 具体的なコード

```js
import { randomUUIDv7, randomUUID } from 'node:crypto';

// UUID v7: 時系列ソート可能（推奨）
const id = randomUUIDv7();
console.log(id);
// 例: "01961234-5678-7abc-8def-0123456789ab"
//      ^^^^^^^^ ← 最初の48bitがUnixミリ秒タイムスタンプ

// UUID v4: 完全ランダム（従来）
const idV4 = randomUUID();

// データベース挿入例
import { createClient } from 'pg';
const client = createClient();

await client.query(
  'INSERT INTO users (id, name) VALUES ($1, $2)',
  [randomUUIDv7(), 'Alice']
);
// v7により挿入順がインデックスと一致し、B-treeのページ分断を回避
```

> **注意:** システムクロックが単調増加でない場合（時刻の巻き戻し）、UUID v7の時系列保証は崩れる。

### ③ なぜ効果的か

PostgreSQL・MySQL・MongoDBなどのUUID主キー運用でホットスポット問題が解消され、書き込み集中時のI/Oコストが下がる。`uuid`パッケージへの依存が不要になり、供給チェーンリスクを1つ削減できる。

---

## 5. npmエコシステムの脅威状況 — Miasma / Mini Shai-Hulud サプライチェーン攻撃

**出典:** https://unit42.paloaltonetworks.com/monitoring-npm-supply-chain-attacks/ (Updated June 2, 2026)

### ① 目的・問題意識

2025年9月の「Shai-Hulud」ワームを起点として、npmエコシステムを標的にした高度なサプライチェーン攻撃が継続的に進化している。攻撃者はCI/CDパイプラインへの永続化を目指し、盗んだトークンで自動的に正規パッケージを汚染・再公開するワーム型マルウェアを使用する。

### ② 具体的な攻撃事例と技術詳細

#### Mini Shai-Hulud — TanStack攻撃（2026年5月）

2026年5月11日 19:20〜19:26 UTC の6分間で、`@tanstack`スコープ配下42パッケージの84バージョンが汚染された。

**攻撃チェーン（3段階のGitHub Actions脆弱性悪用）**

```
1. Fork & Cache Poisoning
   └─ 攻撃者がTanStack/routerをフォーク（zblgg/configurationと改名）
   └─ forked PRでGitHub Actionsキャッシュ（pnpmストア）を汚染

2. OIDCトークン奪取
   └─ 攻撃者管理のバイナリが /proc/<pid>/mem からGitHub Actions
      OIDCトークンを直接読み取り

3. 正規パイプラインによる悪意あるパッケージ公開
   └─ 盗んだOIDCトークンを使い、TanStackの正規リリース
      パイプラインとして npm publish を実行（npmパスワード不要）
```

**悪意あるペイロードの構造**

```
router_init.js（新規追加ファイル）
├── 2.3MBの難読化JavaScriptペイロード
├── 多段デコード（eval() + ROTベースエンコーディング）
├── デーモン化（macOS: LaunchAgent / Linux: systemd）
└── 窃取対象:
    ├── CI/CDトークン (GitHub Actions OIDC, GitLab, CircleCI)
    ├── クラウド認証情報 (AWS IMDSv2, GCP, Azure)
    ├── Kubernetes サービスアカウントトークン
    ├── HashiCorp Vault トークン
    └── npmレジストリトークン
```

**データ漏洩の3チャンネル（冗長化）**

```
1. タイポスクワットドメイン: git-tanstack[.]com
2. 分散型メッセンジャー: Sessionネットワーク
3. GitHub APIデッドドロップ: 盗んだトークンを使ったGist/Issue
```

**永続化バックドア**

```bash
# macOS: LaunchAgent として登録
~/Library/LaunchAgents/com.gh-token-monitor.plist

# Linux: systemd サービスとして登録
~/.config/systemd/user/gh-token-monitor.service

# 動作: 60秒ごとにGitHubポーリング
# トークン無効化を検知するとrm -rf ~/. を実行（24時間後自動終了）
```

**npm依存グラフによるワーム伝播**

```bash
# npmはGit依存のlifecycleスクリプト(prepare)を自動実行する
# つまり package.json で以下のように依存するだけで感染が広がる
{
  "dependencies": {
    "@tanstack/router": "git+https://github.com/org/router.git"
  }
}
# → npm install 実行時に prepare スクリプト（= マルウェア）が実行される
```

#### Miasma — Red Hat npm攻撃（2026年6月1日）

`@redhat-cloud-services`スコープ配下32パッケージ・96バージョンが汚染（週間DL約117,000回）。

**侵入経路**

```
1. Red Hat従業員のGitHubアカウントがインフォスティーラーに漏洩
   （Whiteintelが4月13日・5月15日のログを検知）
2. 漏洩したセッションクッキーでCI/CDワークフローを侵害
3. コードレビューをバイパスし、preinstallスクリプトを注入
```

**マルウェアの特徴**

```js
// package.json に追加された悪意あるpreinstallフック
{
  "scripts": {
    "preinstall": "node index.js"
  }
}
// → npm install 実行時に自動でindex.jsが実行される
```

**Miasma（Mini Shai-Hulud派生）の追加機能**

Mini Shai-Hulud（TeamPCPが2026年5月12日にオープンソース化）をベースに、GCP・Azureのクラウドアイデンティティ窃取コレクターが追加されている。

### ③ 緩和策

```bash
# 1. npm auditの実行
npm audit

# 2. package-lock.json / lockfileの検証（整合性チェック）
npm ci --ignore-scripts  # postinstall/preinstallスクリプトを無効化

# 3. 信頼できるパッケージのみ許可（Provenance検証）
npm install --require-provenance @scope/package

# 4. GitHub Actionsのワークフロー権限を最小化
# .github/workflows/release.yml
permissions:
  contents: read
  id-token: write   # 必要最小限のOIDCスコープのみ許可

# 5. npm tokenのrotation
npm token revoke <token>
npm token create --read-only  # 読み取り専用トークンを使い分ける
```

### ③ なぜ効果的（な防御策）か

従来の「悪意あるパッケージを作る」手法と異なり、これらの攻撃は**正規のリリースパイプライン自体を乗っ取る**。攻撃者が正規のOIDCトークンで公開するため、パッケージの署名や公開者チェックだけでは検出が困難。`npm ci --ignore-scripts`による自動スクリプト実行の無効化と、SLSA/Provenanceによる供給元検証が有効な緩和策となる。

---

## 6. Node.js 24.16.0 (LTS "Krypton") リリース

**出典:** https://nodejs.org/en/blog/release/v24.16.0  
**リリース日:** 2026年5月21日

### ① 目的・問題意識

Node.js 24系のLTS（長期サポート）最新版。26系で導入された新機能の多くをバックポートし、本番環境向けのLTSユーザーに提供する。

### ② 主要な変更点とコード

**`util.styleText()` 16進数カラー対応（新機能）**

```js
const { styleText } = require('util');

// 16進数カラーコード指定
console.log(styleText('#FF5733', 'カスタムオレンジテキスト'));
console.log(styleText('#2ECC71', '緑色テキスト'));

// 複数スタイルの組み合わせ
console.log(styleText(['bold', '#FF5733'], '太字のオレンジ'));
```

**`randomUUIDv7()` のバックポート（26.1から）**

```js
const { randomUUIDv7 } = require('crypto');
const id = randomUUIDv7();
// 時系列ソート可能なUUID v7
```

**`fs.stat()` に `signal` オプション**

```js
const fs = require('fs');
const { AbortController } = require('abort-controller');

const ac = new AbortController();
setTimeout(() => ac.abort(), 2000); // 2秒でキャンセル

fs.stat('/slow/network/path', { signal: ac.signal }, (err, stats) => {
  if (err?.code === 'ABORT_ERR') {
    console.log('stat操作がキャンセルされました');
  }
});
```

**テストランナーのランダム順実行**

```bash
# テスト実行順をランダム化してフレーキーテストを発見
node --test --test-randomize-order test/*.js

# シードを固定して再現
node --test --test-randomize-order --test-randomize-seed=12345 test/*.js
```

**`http.IncomingMessage.signal` の追加**

```js
const http = require('http');

const server = http.createServer((req, res) => {
  // クライアントがコネクションを切断したことを検知
  req.signal.addEventListener('abort', () => {
    console.log('クライアントが切断しました — 処理を中断');
    // 重い処理を中断するなど
  });
  
  res.end('OK');
});
```

### ③ なぜ効果的か

LTS版への機能バックポートにより、本番環境を安定させながら最新機能（UUID v7、カラー出力強化、AbortSignal統合）を享受できる。特に`req.signal`はSSE/ストリーミングレスポンスのリソースリーク防止に直結する。

---

## まとめ

| トピック | 重要度 | キーワード |
|----------|--------|------------|
| replacements.fyi | ★★★ | npm依存削減、e18e、ネイティブAPI移行 |
| Node.js 26.0.0 | ★★★ | Temporal API、V8 14.6、Undici 8 |
| Node.js 26.1.0 `node:ffi` | ★★★ | ネイティブライブラリ連携、FFI、dlopen |
| `randomUUIDv7()` | ★★ | 時系列UUID、DB最適化 |
| Miasma/Mini Shai-Hulud | ★★★ | サプライチェーン攻撃、CI/CD汚染 |
| Node.js 24.16.0 LTS | ★★ | styleText 16進数、LTSバックポート |
