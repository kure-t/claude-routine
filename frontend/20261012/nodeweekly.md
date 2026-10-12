# Node Weekly Issue 642 まとめ (September 24, 2026)

> 出典: [Node Weekly Issue 642](https://nodeweekly.com/issues/642)
> 取得上の注意: nodeweekly.com への直接アクセスは不可のため WebSearch の結果のみで構成。実行日(2026-10-12)時点で検索から確認できた最大番号は #642（#643 以降は検索に出ず未確認）。記事本文を直接開けていないため、各項目の「コード」は公式ドキュメント/PR 記載の挙動と検索で得た情報に基づく**説明用の例**であり、原文の完全な転載ではない。スポンサー記事は除外。

---

## 1. 12 Node built-ins that replace npm packages（メイン記事）

**概要:** axios・dotenv・nodemon・chalk などの npm パッケージを Node 組み込み機能で置き換える手引き。各機能の Node 24 LTS での安定度と、依存を残した方がよい落とし穴も併記している（12項目すべての一覧は本文未確認）。

### 1-1. グローバル `fetch`（axios の代替）
- ① 目的: HTTP クライアントのために axios を入れる必要を減らす。Node 18 以降グローバルで利用可能（内部は undici）。
- ② 例:
```js
const res = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ name: 'alice' }),
});
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const data = await res.json();
```
- ③ 効果: 依存ゼロで標準 API。ただし axios の interceptors 等の高度機能は無く、4xx/5xx で例外を投げない点に注意。

### 1-2. `--env-file`（dotenv の代替）
- ① `.env` 読み込みのためだけの dotenv を不要にする。
- ② 例: `node --env-file=.env app.js`（ファイルが無くても良い場合は `--env-file-if-exists=.env`）
```js
console.log(process.env.DATABASE_URL);
```
- ③ 効果: 起動時にロードされるためコード変更不要。注意: 値内の変数展開はされず、ファイル欠落時は既定でエラー。

### 1-3. `--watch`（nodemon の代替）
- ① 変更時の自動再起動。
- ② 例: `node --watch server.js` / `node --watch-path=./src server.js`
- ③ 効果: 追加ツール不要。既定ではエントリと import されたモジュールを監視。`--watch-path` は macOS/Windows のみ対応との情報あり。

### 1-4. `util.styleText`（chalk の代替）
- ① ターミナル色付けの軽量化。
- ② 例:
```js
import { styleText } from 'node:util';
console.log(styleText('red', 'error'), styleText(['bold', 'green'], 'ok'));
```
- ③ 効果: 基本的な装飾は標準で足りる。chalk の豊富な API（rgb 等）は無い。

### 1-5. `node:test`（テストランナー）
- ② 例:
```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
test('adds', () => assert.equal(1 + 1, 2));
```
実行: `node --test`。注意: モジュール全体の `mock.module()` は実験的との情報。

※ 残りの項目は本文を確認できず未収録。

---

## 2. `util.markPromiseAsHandled()`（Node 26.10 で追加）
- **リンク:** [PR #65805](https://github.com/nodejs/node/pull/65805) / [docs](https://nodejs.org/api/util.html) / 文言修正 [PR #66206](https://github.com/nodejs/node/pull/66206)
- ① 問題意識: unhandled rejection の発生を避けるためだけに `.catch(() => {})` を付ける回避策が横行している。
- ② 技術詳細: 指定した Promise をハンドル済みとして印を付け、その Promise の未処理 rejection が報告されなくなる。`util.throttle()` が rejected 呼び出しにこの挙動を使う。
```js
import { markPromiseAsHandled } from 'node:util';
const p = doWork();          // 後で await するかもしれない Promise
markPromiseAsHandled(p);     // 従来の p.catch(() => {}) の代替
```
- ③ 効果: 意図を明示でき、握りつぶし用の空 catch を排除できる。（例のコードは説明用）

## 3. `--process-timeout=N`（提案/PR 段階）
- **リンク:** [PR #66138](https://github.com/nodejs/node/pull/66138)
- ① 問題意識: OS/CI のタイムアウトは環境差があり、`setTimeout` はイベントループが空かないと発火せず、ビジーループやスレッドブロックを検知できない。
- ② 技術詳細: 期限超過時に `Process timed out after 10s (--process-timeout)` を出力し、メインスレッドのスタックトレースを含めて終了コード **124** で終了。制約: 猶予期間は固定 2 秒（その間メインスレッドが CPU を得られないと診断失敗）、Windows では pid 再利用時に最大 60 秒残る場合あり。
```
node --process-timeout=10 script.js   # 10秒で強制終了、exit code 124
```
- ③ 効果: ハングしたプロセスの原因（スタック）が分かる確実な上限時間を提供。

## 4. その他の見出し（本文未確認）
- safer `eval()`（安全な eval の提案）
- Node 26.10 リリース
- Node 27 で Intel Mac の公式ビルド終了予定
- pnpm 12.6: ワークスペース重複排除、node_modules 移動対応

---
未確認事項: 上記 4 の詳細、1 の残り項目、他セクション記事。デモ/動画リンクは確認対象外。
