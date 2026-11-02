# Node Weekly Issue #645 (2026-10-29) まとめ

「Node security releases to fix a Brotli flaw」
https://nodeweekly.com/issues/645

> 注記: 実行環境から nodeweekly.com・redmonk.com・electronjs.org 等は名前解決できず直接取得不可。WebSearch の要約と、取得できた GitHub (reylo / opentelemetry-js) の内容に基づく。取得できなかった記事は詳細が限定的（完全なコード等は未確認）。Sponsor 記事は除外。

---

## 1. Node.js セキュリティリリース (v26.12.1 / v24.22.1 / v22.23.4)

### ① 目的・問題意識
2026-10-21 に 10 件の CVE を修正。最重要は同梱 Brotli デコーダの境界外書き込み **CVE-2026-111497**（High、Google の CVSS 4.0 は 9.2 Critical、通称 Node2Shell）。悪意ある Brotli データを `node:zlib` 等で展開するアプリで RCE の可能性。CISA KEV に 10/27 登録、実際に悪用されている。回避策は無い。

### ② 技術詳細
| CVE | 重大度 | 内容 | 対象 |
|---|---|---|---|
| CVE-2026-111497 | High | Brotli デコーダ OOB write → RCE | 22/24/26 |
| CVE-2026-105482 | High | `dns.resolve` が NUL 文字を含む名前を切り詰める | 22/24/26 |
| CVE-2026-105483 | High | 全角数字 IP による TLS ホスト名検証バイパス | 24/26 |
| CVE-2026-106069 | High | IP に緩く解決されるホストでの TLS ホスト名バイパス | 22/24/26 |
| CVE-2026-105481 | Medium | HTTP keep-alive | |
| CVE-2026-106072 | Medium | TLS SAN IPv4 | |
| CVE-2026-106074 | Medium | `randomUUID()`/`randomInt()` のキャッシュが startup snapshot / SEA に保存される | |
| CVE-2026-106070 | Low | ZipFile | 26 のみ |
| CVE-2026-106071 / 106073 | Low | Permission Model | 26 のみ / 全ライン |

修正は c-ares, OpenSSL, SQLite, Undici, eMudhra CA 証明書などの同梱依存も含む。ライン別修正数: 26.x=10, 24.x=8, 22.x=7。Node 20 は 2026-04-30 EOL で修正なし。Brotli は PHP 等も共有依存で、Node 更新では他製品は直らない。

対応（コードではなく運用）:
```bash
node -v   # 26.12.1 / 24.22.1 / 22.23.4 以上か確認（現行最新 26.13.0）
```
本番・CI・Docker イメージ・Electron など「実際に動く全ての Node」を洗い出して更新する。

### ③ 効果
境界外書き込みは攻撃者制御データの展開だけでメモリ破壊に至るため、アプリ側の緩和策が効かず、ランタイム更新が唯一の対策になる。

---

## 2. Node 26 が LTS に (26.13.0 "Lithium")

### ①
Node 26 が 2026-10-28 に LTS 昇格。2029 年 4 月までサポート。Node 24 はメンテナンスへ、Node 22 は 2027-04 までメンテナンス LTS。
### ②
Node 26 は Temporal 搭載（26.0）、FFI 追加（26.1、26.9 で既定有効）など。移行は `nvm install 26` 等で行い、自アプリでベンチ・互換性を実測して判断する（参考: DEV Community "Node 26 is LTS today"）。
### ③
本番は LTS 以降が推奨で、CVE 修正の最新ラインでもあるため、今回の脆弱性対応と併せて移行しやすい。

---

## 3. Reylo — 型付き `.sql` インポート (Zod 作者 colinhacks)

https://github.com/colinhacks/reylo

### ① 目的
SQL を TypeScript 文字列や codegen に頼らず、`.sql` ファイルを import するだけで引数・結果行の型を推論する。生成ファイルも watcher も不要。TypeScript 7.1 beta の **content mappers**（tsc が `.sql` 等の外部ファイルを型付けするパッケージ呼び出し機構）を利用。

### ② 具体例
要件: TypeScript 7.1 beta、Node 24+、稼働中の開発 DB（Postgres / MySQL 8.0.22+ / SQLite / DuckDB）。

`tsconfig.json`:
```json
{
  "compilerOptions": { },
  "contentMappers": [
    { "package": "reylo", "extensions": [".sql", ".reylo.yaml", ".reylo.yml"] }
  ]
}
```
型チェック（このフラグが必須）:
```bash
tsc --noEmit --runExternalCode
```
`DATABASE_URL` を環境変数/.env に設定（スキームで DB 選択）。実行時:
```bash
node --env-file=.env --import reylo/register main.ts
```
（Bun, Vite, Next.js, esbuild, webpack, Rspack, Cloudflare Workers, Jest 用プラグインあり）

```sql
-- get_user.sql
select id, name, email from users where id = :id::int
```
```ts
import getUser from "./get_user.sql";

const user = await getUser.run(db, { id: 1 });
// { id: number; name: string; email: string | null } | null
```
- パラメータは `:name`。型が決まらない場合は `::int` のようにキャスト。
- 戻り値: unique キー一致/`limit 1` → `Row | null`、group by 無しの集約や 1 行 `insert ... returning` → `Row`、returning 無しの insert/update/delete → `void`、その他 → `Row[]`。
- 他に `.runSync()`, `.try()`, `.with()`, `.sql`, `.raw`, `.source`。全クエリは Node の `reylo:query` tracing channel に発行。
- ドライバ: pg, postgres.js, mysql2, better-sqlite3, node:sqlite, libSQL/Turso, Cloudflare D1, @duckdb/node-api、これらの上の Drizzle。

### ③ なぜ効果的か
開発 DB に対してクエリを prepare（実行はしない）し、列と引数の型を DB から読み戻すので、スキーマと常に一致する。null 可否は schema・join・where から判定、unique 一致/集約でカーディナリティ推定、enum や check 制約は union 型、JSON も形状を保持。

関連: sqlfu の PR #157（definitions.sql から DB 不要で型付け）、Lojhan/typed-sql（TS7 ネイティブ推論）。ただし content mapper は外部ファイルのみ対象で `.ts` 内 tagged template は型付けできない。

---

## 4. OpenTelemetry JS 3.0

https://github.com/open-telemetry/opentelemetry-js/releases/tag/v3.0.0 / 移行ガイド: `doc/3.x/migration-guide.md`

### ① 目的
SDK 3.0.0 / 0.300.0 / API 1.10.0（2026-10-20）。ESM/CJS デュアルビルドと厳格な exports で整理し、トレース系パッケージを統合。2.x は 1 年サポート。併せて Node.js org に最小 OTel 実装を試作する `microtel` リポジトリ（将来 Node 本体への取り込みが目標）。

### ② 破壊的変更と移行
- Node.js >= 22.15.0（18/20 切り捨て）。
- exports 厳格化でディープインポート（`@opentelemetry/core/build/src/...`）は `ERR_PACKAGE_PATH_NOT_EXPORTED`。`package.json` の import ではなく `SDK_INFO`（`@opentelemetry/core`）を使う。
- バンドラ: Webpack 5（enhanced-resolve 5.8.2+）、Jest 29.4+、Metro 0.81.3+、Parcel は packageExports 有効化。
- `sdk-trace-base/node/web` → `@opentelemetry/sdk-trace` に統合。`BasicTracerProvider`/`WebTracerProvider` → `TracerProvider`。`register()` は廃止:
```ts
import { trace, context, propagation } from '@opentelemetry/api';
trace.setGlobalTracerProvider(provider);
context.setGlobalContextManager(new AsyncLocalStorageContextManager());
propagation.setGlobalPropagator(new W3CTraceContextPropagator());
```
- SpanProcessor は options 形式: `new SimpleSpanProcessor({ exporter })`。
- `@opentelemetry/api-logs` 廃止、`@opentelemetry/api` に統合。`setGlobalLoggerProvider` は boolean を返す → `logs.getLoggerProvider()`。
- 複雑な属性値（オブジェクト・null 等）がスパンで保持される（Resource 等では警告して破棄、メトリクスは非検証）。
- 削除: `propagator-jaeger`（W3C へ。移行中は `CompositePropagator`）、`exporter-jaeger`（OTLP へ）、`shim-opentracing`/`shim-opencensus`、`getTimeOrigin`/`otperformance`/`_globalThis`/`unrefTimer`（→ `performance.timeOrigin`/`performance`/`globalThis`/`timer.unref()`）、`AsyncHooksContextManager`（→ `AsyncLocalStorageContextManager`）、`forceFlushTimeoutMillis`（→ `forceFlush({ timeoutMillis })`）。
- `sdk-node`: 単数オプションが配列化（`spanProcessors`, `metricReaders`, `logRecordProcessors`）、名前空間再エクスポート廃止、`jaeger` propagator 名廃止。環境変数設定は sdk-node 経由。
- `exporter-prometheus` は既定 localhost バインド、`host`/`port` をコンストラクタで指定。`LoggerConfig.disabled` → `enabled`（意味反転）。
- 3.x は `@opentelemetry/api >=1.10.0 <1.11.0` が必須。semantic-conventions は 1.x のまま。
- @vercel/otel など一部は 3.x 未対応で Next.js 利用者は待ち。

### ③ 効果
パッケージ統合と exports 厳格化でバンドル挙動が予測可能になり、API の肥大化が減る。

---

## 5. JavaScript ランタイムのモノカルチャー問題 (RedMonk, 2026-10-27)

https://redmonk.com/kholterhoff/2026/10/27/js-runtime-monoculture/

### ① 問題意識
Deno チームが Cloudflare に参加し、Ryan Dahl によれば Deno ランタイムの開発は 1 年後に停止。汎用ランタイムは Node.js と Bun に絞られ、workerd は主に Cloudflare 向け。Deno は財団化を模索、コミュニティフォーク「Ende」も登場。
### ② 論点
Deno は権限ベースセキュリティを主流にし、Node の TS 対応・ツール改善を促した。Linux のように独立実装が多数ある状態と対比。エンジン面でも Node/Deno/workerd は V8、Bun は JavaScriptCore で、ブラウザで起きた寡占がサーバに及ぶ懸念。
### ③ 示唆
競争が標準化と機能進化を牽引してきたため、寡占は進化速度と相互運用性のリスクになる。

---

## 6. Electron 45

https://www.electronjs.org/blog/electron-45-0

### ① / ② 内容
2026-10-20 リリース。Chromium 156、V8 15.6、Node 24.22（45.1.0 以降 24.22.1 = Brotli 修正入り）。45.2.0 は 10/30。
- ASAR 内の大きなファイルのコールド読み込みが最大 100 倍高速。
- 破壊的変更: `nodeIntegrationInWorker: true` の iframe 由来 Worker は `nodeIntegrationInSubFrames` も有効な場合のみ Node 統合。画面/ウィンドウ/タブキャプチャ要求は `session.setPermissionRequestHandler()` に `media` ではなく `display-capture` として届く。サンドボックス preload から Node モジュール除去、32bit ビルド廃止（二次情報）。
- 45.0.0 のサポート終了は 2027-04-27。
### ③
Electron アプリは同梱 Node の脆弱性も継承するため、45.1.0 以降へ更新が必要。

---

### 参考ソース
- https://nodeweekly.com/issues/645
- https://tuxcare.com/blog/node-js-10-cves/
- https://www.konfirmity.com/advisories/nodejs-october-2026-security-releases
- https://github.com/colinhacks/reylo
- https://github.com/iterate/sqlfu/pull/157
- https://github.com/open-telemetry/opentelemetry-js/releases/tag/v3.0.0
- https://releases.electronjs.org/release/v45.0.0
