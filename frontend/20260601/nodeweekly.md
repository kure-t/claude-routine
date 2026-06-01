# Node Weekly Issue 626 (May 28, 2026) まとめ

> 原文: https://nodeweekly.com/issues/626

---

## 1. The Production Playbook for Node.js Stream Leaks

**出典:** [Frontend Masters Blog](https://frontendmasters.com/blog/the-production-playbook-for-node-js-stream-leaks/)  
**著者:** Durgesh Rajubhai Pawar（2026-05-26）

### ① 目的・問題意識

Node.js のストリームは、ローカルの高速マシン・小データセット・切断しないクライアントでテストすると問題を起こさない。しかし本番環境ではユーザーがタブを閉じたり、タイムアウトが起きたり、低速ネットワークで長時間つながりっぱなしになる。この記事は、**テストやコードレビューをすり抜けるが本番トラフィックで確実に壊れる5つのリークパターン**とその撃退法（5ルール）を解説する。

これは Frontend Masters Blog 上の2部構成シリーズの第2弾。第1弾（"Your Node.js Streams Aren't Backpressuring. They're Silently Eating Your Memory."）では、`highWaterMark` はメモリ上限ではなく「advisory threshold（推奨閾値）」に過ぎないというコアメンタルモデル、`pipeline()` への移行、async/await がデータ量問題を解決しない点を解説。

---

### ② 具体的な技術詳細

#### 基本前提：`pipeline()` と `AbortController` の組み合わせ

```javascript
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';

// ✅ 推奨パターン
const ac = new AbortController();
res.on('close', () => ac.abort());  // クライアント切断でキャンセル

await pipeline(
  fetchRows(cursor),   // Readable: DBカーソルなど
  formatCSV,           // Transform
  res,                 // Writable: HTTPレスポンス
  { signal: ac.signal }
);
```

#### リークパターン1：`.pipe()` とクライアント切断

**問題:** `.pipe()` は `finish` イベントでのみティアダウンをトリガーする。ユーザーがブラウザタブを閉じると `close` が発火するが `finish` は発火しない → 上流のDBクエリ・Transformが動き続けてメモリが増大。

```javascript
// ❌ 危険なパターン
const readable = db.query('SELECT * FROM huge_table');
readable.pipe(csvTransform).pipe(res);
// ユーザーが切断しても上流は動き続ける

// ✅ 修正: pipeline + AbortController
const ac = new AbortController();
res.on('close', () => ac.abort());
await pipeline(readable, csvTransform, res, { signal: ac.signal });
```

#### リークパターン2：タイムアウトが残すリーク

**問題:** `res.setTimeout()` でタイムアウト後に `res.end()` を呼ぶと、`end()` はwritableの片側を gracefully close するだけ。`pipeline()` はこれをエラーとみなさず上流 fetch が走り続ける。

```javascript
// ❌ 問題のある実装
res.setTimeout(5000, () => {
  res.end();  // pipeline() は finish とみなしてしまう
});
await pipeline(fetchFromExternalApi(), res);

// ✅ 修正: AbortSignalでタイムアウト
const ac = new AbortController();
const timer = setTimeout(() => ac.abort(new Error('timeout')), 5000);
try {
  await pipeline(fetchFromExternalApi(), res, { signal: ac.signal });
} finally {
  clearTimeout(timer);
}
```

#### リークパターン3：外部ハンドル（サイドチャネルリソース）

**問題:** Transform がキャッシュカーソル・一時ファイル・子プロセス・サイドチャネルソケットを持つ場合、`pipeline()` がストリームを abort/destroy しても外部ハンドルは自動クリーンアップされない。

```javascript
// ❌ 問題のある Transform
class CsvTransform extends Transform {
  constructor() {
    super();
    this.tempFile = fs.openSync('/tmp/buffer', 'w+');  // 外部ハンドル
  }
  _transform(chunk, enc, cb) { /* ... */ cb(); }
  // _destroy() がないので tempFile はリークする
}

// ✅ 修正: _destroy() で外部リソースを閉じる
class CsvTransform extends Transform {
  constructor() {
    super();
    this.tempFile = fs.openSync('/tmp/buffer', 'w+');
  }
  _transform(chunk, enc, cb) { /* ... */ cb(); }
  _destroy(err, cb) {
    fs.closeSync(this.tempFile);  // abort パスでも必ず実行される
    cb(err);
  }
}
```

#### リークパターン4：`finish` だけに依存したクリーンアップ

**問題:** `finish` イベントのみにクリーンアップを置くと、`destroy()` パス（abort や error）では `finish` が発火しない場合があり、一時ファイルやプール接続が残留する。

```javascript
// ❌ 問題: 'finish' だけを監視
writable.on('finish', () => {
  dbConnection.release();  // destroy() パスではスキップされる場合がある
});

// ✅ 修正: 'close' を使う（finish と error 両方の後に必ず発火）
writable.on('close', () => {
  dbConnection.release();
});
```

#### リークパターン5：DBコネクションとクライアントネットワーク速度の結合

**問題:** HTTPレスポンスが終わったらDBコネクションを解放するという実装では、低速（3G）ユーザーが5分かけてダウンロードすると5分間DBワーカーを占有する。

```javascript
// ❌ 問題: DB接続の生存期間がクライアントのネットワーク速度に依存
const stream = db.query('SELECT * FROM exports').stream();
pipeline(stream, res).then(() => db.releaseConnection());

// ✅ 修正: バッファリングして先に接続を解放
async function* bufferedQuery(db) {
  const rows = await db.query('SELECT * FROM exports');
  db.releaseConnection();  // データ取得完了後に即リリース
  for (const row of rows) yield formatRow(row);
}
await pipeline(bufferedQuery(db), res);
```

---

### ③ なぜ効果的か

- `pipeline()` + `AbortController` を組み合わせることで、**クライアント切断・タイムアウト・エラーのすべてのパス**でストリームチェーン全体がクリーンアップされる
- `_destroy()` フックは destroy/abort パスでも確実に呼ばれるため、外部リソースのクリーンアップに最適
- `'close'` イベントは `finish` と `error` の両方の後に発火するため、ライフサイクル管理のアンカーポイントとして信頼できる
- **5ルールのプレイブック:**
  1. `.pipe()` より `pipeline()` を使う
  2. `pipeline()` の返すbooleanをチェックする
  3. 自分で作ったものは自分で destroy する（`_destroy()` フック）
  4. 本番前に制限されたヒープでプロファイリングする
  5. バックプレッシャーテストをCIに組み込み、PRマージ前に回帰を検知する

---

## 2. Node.js 26.2.0 リリース（2026-05-20）

**出典:** [Node.js Blog](https://nodejs.org/en/blog/release/v26.2.0)

### ① 目的・問題意識

Node.js v26.1.0 の後継マイナーリリース。`stream.compose` の安定化、Temporal API の `fs.Stats` への統合、HTTP 1xx ステータスコードサポート、暗号強化が主なテーマ。

### ② 具体的な技術詳細

#### `stream.compose` の安定化（Stable）

`stream.compose` はストリームを合成（compose）するユーティリティ。Node.js 26.2.0 で正式に Stable に昇格した。

```javascript
import { compose, Readable, Transform } from 'node:stream';

// 複数の Transform を1つのストリームとして合成
const pipeline = compose(
  new Transform({
    transform(chunk, enc, cb) {
      cb(null, chunk.toString().toUpperCase());
    }
  }),
  new Transform({
    transform(chunk, enc, cb) {
      cb(null, `[${chunk}]`);
    }
  })
);

Readable.from(['hello', 'world'])
  .pipe(pipeline)
  .pipe(process.stdout);
// 出力: [HELLO][WORLD]
```

#### `fs.Stats` への `Temporal.Instant` サポート（SEMVER-MINOR）

ファイルのタイムスタンプ（atime, mtime, ctime, birthtimeなど）を `Temporal.Instant` で取得できるようになった。ナノ秒精度でありながら、モダンな Temporal API で扱える。

```javascript
import { statSync } from 'node:fs';

const stats = statSync('file.txt');

// 従来の Date オブジェクト（ミリ秒精度）
console.log(stats.mtime);             // Date object
console.log(stats.mtimeMs);           // milliseconds since epoch

// 新: Temporal.Instant（ナノ秒精度）
const instant = stats.mtimeNs;       // BigInt nanoseconds (従来)
// Temporal.Instant は Stats オブジェクトに統合されており、
// Node.js 26.0 で有効化された Temporal API との統合が完成した

// BigIntStats（ナノ秒精度）
const bigStats = statSync('file.txt', { bigint: true });
console.log(bigStats.mtimeNs);        // 1748476800000000000n など
```

#### HTTP 1xx ステータスコードサポート（SEMVER-MINOR）

`res.writeInformation()` で任意の 1xx ステータスコードを送信できるようになった。103 Early Hints などの実装が容易に。

```javascript
import { createServer } from 'node:http';

const server = createServer((req, res) => {
  // 103 Early Hints: ブラウザにCSSのプリロードを指示
  res.writeInformation(103, {
    'Link': '</style.css>; rel=preload; as=style'
  });
  res.end('Hello World');
});

server.listen(3000);
```

#### 主な依存関係アップデート

| パッケージ | バージョン |
|-----------|-----------|
| undici | 8.3.0 |
| corepack | 0.35.0 |
| sqlite | 3.53.1 |
| simdjson | 4.6.4 |

### ③ なぜ効果的か

- `stream.compose` の安定化により、複数の Transform を再利用可能なユニットとして合成できる（`pipeline()` との組み合わせで堅牢なストリーム設計が可能）
- `Temporal.Instant` の `fs.Stats` 統合により、ファイル操作と時刻比較が Temporal の型安全で一貫したAPIで扱える（タイムゾーン処理バグが減少）
- `writeInformation()` で 103 Early Hints を実装することで、サーバーサイドレンダリング時のリソースプリロードによるページ表示速度改善が標準的なAPIで実現可能

---

## 3. Honker — SQLite に Postgres の NOTIFY/LISTEN セマンティクスを

**出典:** [GitHub: russellromney/honker](https://github.com/russellromney/honker) / [honker.dev](https://honker.dev/)

### ① 目的・問題意識

SQLite は組み込み・サーバーレス環境でのシンプルなデータベースとして広く使われているが、Postgres の `NOTIFY`/`LISTEN` のようなリアルタイムイベント通知機能がない。従来はポーリング（定期的にDBをクエリ）や別途メッセージブローカー（Redis、RabbitMQ等）が必要だった。

**Honker** は SQLite の拡張（SQLite Extension + 各言語バインディング）として、ポーリングもデーモンも不要で**1ファイル（.db）だけでリアルタイムイベント、永続的Pub/Sub、タスクキューを実現する。**

### ② 具体的な技術詳細

#### アーキテクチャ

- **WALモード（Write-Ahead Log）** を活用し、別プロセスのコミットを `PRAGMA data_version` を1msごとにポーリングして検知
- コミット検知後、サブスクライバーへ bounded channel 経由でイベントを fan-out
- クロスプロセス通知レイテンシ: **~0.7ms p50**（M-series Mac計測）

#### 3つのプリミティブ

| プリミティブ | 説明 |
|------------|------|
| `notify` | 一時的なPub/Sub（Postgresの `NOTIFY` 相当） |
| `stream` | 永続的なPub/Sub（コンシューマーごとのオフセット管理、リプレイ可能） |
| `queue` | 少なくとも1回（at-least-once）のワークキュー（リトライ・デッドレター対応） |

#### Node.js コード例

```javascript
const { open } = require('@russellthehippo/honker-node');
const db = open('app.db');

// アトミックトランザクション: DBへの書き込みと通知を一緒にコミット
const tx = db.transaction();
tx.execute('INSERT INTO orders (id) VALUES (?)', [42]);
tx.notify('orders', { id: 42 });
tx.commit();

// WAL更新イベントを監視してリアルタイム通知を受け取る
const ev = db.updateEvents();
let lastSeen = 0;
while (running) {
  await ev.next();  // WALへのコミットで wake up
  const rows = db.query(
    'SELECT id, payload FROM _honker_notifications WHERE id > ? ORDER BY id',
    [lastSeen]
  );
  for (const row of rows) {
    handle(JSON.parse(row.payload));
    lastSeen = row.id;
  }
}
```

#### Python コード例（参考）

```python
import honker

db = honker.open("app.db")

# ストリーム（永続Pub/Sub）
stream = db.stream("user-events")

# パブリッシャー: DBへの書き込みとイベント発行をアトミックに
with db.transaction() as tx:
    tx.execute("UPDATE users SET name=? WHERE id=?", [name, uid])
    stream.publish({"user_id": uid, "change": "name"}, tx=tx)

# サブスクライバー: コンシューマーIDでオフセットを自動追跡
async for event in stream.subscribe(consumer="dashboard"):
    await push_to_browser(event)

# ワークキュー（at-least-once）
emails = db.queue("emails")

with db.transaction() as tx:
    tx.execute("INSERT INTO orders (user_id) VALUES (?)", [42])
    emails.enqueue({"to": "alice@example.com"}, tx=tx)

async for job in emails.claim("worker-1"):
    send(job.payload)
    job.ack()
```

#### SQL 拡張（生SQL）

```sql
.load ./libhonker_ext
SELECT honker_bootstrap();

-- キューにジョブを追加
INSERT INTO _honker_live (queue, payload)
  VALUES ('emails', '{"to":"alice"}');

-- バッチクレーム（ワーカーがジョブを取得）
SELECT honker_claim_batch('emails', 'worker-1', 32, 300);

-- ACK（バッチ）
SELECT honker_ack_batch('[1,2,3]', 'worker-1');

-- スケジューラー（cron形式）
SELECT honker_cron_next_after('0 3 * * *', unixepoch());
SELECT honker_scheduler_register('nightly', 'backups',
  '0 3 * * *', '"go"', 0, NULL);

-- ストリームの発行・読み取り
SELECT honker_stream_publish('orders', 'k', '{"id":42}');
SELECT honker_stream_read_since('orders', 0, 1000);

-- レートリミット
SELECT honker_rate_limit_try('api', 10, 60);
```

#### 対応言語

Python, **Node.js**, Rust, Go, Ruby, Bun, Elixir, C++, .NET/C#, Java/JVM, Kotlin

### ③ なぜ効果的か

- `PRAGMA data_version` を1msポーリングすることで、ファイルベースSQLiteのまま**ブローカーなしのリアルタイム通知**を実現（Redis や RabbitMQ が不要になるケースがある）
- トランザクションとの原子性が保証される（ジョブ追加とビジネス書き込みが同一トランザクション内で完結、ロールバック時はイベントも消える）
- SQLite の1ファイル性を維持しながら、分散システムで必要な primitive（pub/sub, queue, scheduler, rate limit, distributed lock）をすべて実装
- WAL モードにより、複数の読み取りと1つの書き込みを効率的に並列処理できる

---

## 4. Helmet.js 8.2 — Express アプリのセキュリティヘッダー強化

**出典:** [GitHub: helmetjs/helmet](https://github.com/helmetjs/helmet)

### ① 目的・問題意識

Express.js はデフォルトではセキュリティ関連のHTTPレスポンスヘッダーをほとんど設定しない。XSS, クリックジャッキング, MIMEタイプスニッフィング, HTTPS強制などのブラウザセキュリティ機能をHTTPヘッダーで制御するのが現代のベストプラクティスだが、それを手動で設定するのは煩雑でミスも多い。Helmet.js はその設定を1行で行うミドルウェア。

**v8.2.0（2026-05-21）の変更点:**
- `Cross-Origin-Opener-Policy` に `noopener-allow-popups` ディレクティブを追加
- 重複オプション検知時のエラーメッセージを改善

### ② 具体的な技術詳細

#### 基本的な使い方

```typescript
import express from 'express';
import helmet from 'helmet';

const app = express();

// デフォルト設定で13のセキュリティヘッダーを一括適用
app.use(helmet());

app.listen(3000);
```

#### デフォルトで設定される13のヘッダー

| ヘッダー | 目的 |
|---------|------|
| `Content-Security-Policy` | XSS・インジェクション攻撃を許可リストでブロック |
| `Cross-Origin-Opener-Policy` | プロセス分離によるサイドチャネル攻撃対策 |
| `Cross-Origin-Resource-Policy` | 不正なクロスオリジンリソース読み込みをブロック |
| `Origin-Agent-Cluster` | オリジンをプロセスから分離 |
| `Referrer-Policy` | Refererヘッダーで送信する情報を制限 |
| `Strict-Transport-Security` | HTTPSを強制（HSTS） |
| `X-Content-Type-Options` | MIMEタイプスニッフィングを防止 |
| `X-DNS-Prefetch-Control` | DNS プリフェッチを制御 |
| `X-Download-Options` | 安全でないダウンロードを防止（IE8対応） |
| `X-Frame-Options` | クリックジャッキング攻撃を防止 |
| `X-Permitted-Cross-Domain-Policies` | クロスドメインコンテンツ読み込みを制限 |
| `X-Powered-By` | 情報漏洩防止のため削除 |
| `X-XSS-Protection` | ブラウザの不具合のあるXSSフィルターを無効化 |

#### カスタマイズ例

```typescript
// 特定ヘッダーを無効化
app.use(helmet({
  contentSecurityPolicy: false,
  xDownloadOptions: false,
}));

// CSP のカスタマイズ
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      scriptSrc: ["'self'", "example.com"],
      styleSrc: null,  // style-src ディレクティブを無効化
    },
  },
}));
```

#### CSP ノンスを使ったインラインスクリプトの許可

```typescript
import crypto from 'node:crypto';

app.use((req, res, next) => {
  res.locals.cspNonce = crypto.randomBytes(32).toString('hex');
  next();
});

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      scriptSrc: [
        "'self'",
        (req, res) => `'nonce-${res.locals.cspNonce}'`,
      ],
    },
  },
}));

// テンプレート内でノンスを使用
// <script nonce="<%= cspNonce %>">...</script>
```

#### 開発環境での HTTPS 強制を無効化

```typescript
const isDevelopment = app.get('env') === 'development';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      'upgrade-insecure-requests': isDevelopment ? null : [],
    },
  },
}));
```

#### v8.x の主な変更点

| バージョン | 変更内容 |
|-----------|---------|
| v8.2.0 | `Cross-Origin-Opener-Policy` に `noopener-allow-popups` を追加、重複オプション時のエラーメッセージ改善 |
| v8.1.0 | CSP でクォート忘れ（例: `self` → `'self'`）の際のエラーメッセージを改善 |
| v8.0.0 | HSTS の最大有効期間を180日→365日に変更、CSP でクォートなし値はエラーに、Node 16/17サポート終了（要 Node 18以上） |

### ③ なぜ効果的か

- Express は意図的にミニマルな設計のため、セキュリティヘッダーの追加は開発者の責任。Helmet を `app.use(helmet())` 1行で適用するだけで OWASP Top 10 に対応した多層防御が完成する
- 各ヘッダーは個別に無効化・カスタマイズできるため、既存アプリへの段階的導入も容易
- v8.0.0 で HSTS を365日に延長し、CSP のクォートなし値をエラーにすることで、**設定ミスが本番環境に出る前に開発時に検知できる**ようになった
- ノンスベースの CSP により、CDN や外部スクリプトを使いながらも `unsafe-inline` を避けられる

---

## まとめ

Node Weekly #626 のテーマは「**本番ストリームの信頼性、Node.js コアの進化、インフラ削減**」。

- ストリームリークは「テストが通る → 本番で壊れる」典型例。`pipeline()` + `AbortController` + `_destroy()` の三点セットが防衛の基本
- Node.js 26.2.0 で `stream.compose` が安定化し、ストリーム合成がより宣言的に書ける
- Honker はブローカーレスの分散プリミティブを SQLite 1ファイルで実現する新世代ライブラリ
- Helmet.js 8.2 は Express のセキュリティベースラインを1行で担保する定番ミドルウェア
