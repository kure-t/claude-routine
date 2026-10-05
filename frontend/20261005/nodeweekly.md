# Node Weekly 最新号まとめ — Issue #642 (2026-09-24)

- 対象: https://nodeweekly.com/issues/642 「12 Node built-ins that replace npm packages」
- 最新号判定: WebSearch で確認できた最大番号が #642(#643 は未確認。#642 の号で1週休刊の告知あり)
- **取得上の制約**: nodeweekly.com および各リンク先(flaviocopes.com, daily.dev 等)は実行環境のエグレスプロキシで遮断され、WebFetch が不可。WebSearch の要約結果のみが情報源。そのため以下の「検索結果で確認できた内容」と「補足(一般的な Node.js の公開仕様に基づく例。記事原文のコードではない)」を区別して記載する。リンク先の完全なコード/デモの転載はできていない。
- スポンサー記事は除外。

---

## 1. メイン記事: 12 Node built-ins that replace npm packages (Flavio Copes)

元記事: https://flaviocopes.com/node-builtins/

Node 24 LTS を基準に、axios / dotenv / nodemon / chalk / jest / glob 等の npm パッケージを組み込み機能で置き換える方法を、各機能の安定度(stability)と「それでも既存依存を残したほうがよい落とし穴(gotchas)」付きで解説。併せて `util.markPromiseAsHandled()`(`.catch(() => {})` ハックの置換)にも触れる。

### 検索結果から確認できた項目

| # | 置換対象 | 組み込み | 安定度/備考(検索結果より) |
|---|---|---|---|
| 1 | axios, node-fetch | グローバル `fetch()` | Node 21.0.0 以降 stable |
| 2 | nodemon | `node --watch` | 18.11 で導入、22.0.0 以降 stable |
| 3 | dotenv | `--env-file` / `process.loadEnvFile()` | — |
| 4 | chalk, kleur | `util.styleText()` | v20.12.0 導入、v22.17.0 で stable |
| 5 | jest, mocha | `node:test` + `node:assert` | Node 20.0.0 以降 stable |
| 6 | better-sqlite3 | `node:sqlite` | Node 24.15.0 で Release Candidate (1.2) |
| 7 | glob | `fs.glob` / `fs.globSync` | (検索結果で置換対象として言及) |

残り(12個中の他の項目)は検索結果から特定できなかった。

### 各テクニック詳細

#### ① fetch(axios / node-fetch の置換)
- **目的**: HTTP クライアント依存を無くす。
- **コード(補足例)**:
```js
const res = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ name: 'Ada' }),
  signal: AbortSignal.timeout(5000),
});
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const data = await res.json();
```
- **効果的な理由**: undici ベースで標準 Web API のためブラウザと同一コード。タイムアウトは `AbortSignal.timeout`。
- **Gotcha(一般論)**: axios と違い 4xx/5xx で throw しない(`res.ok` 確認が必要)。インターセプタや自動リトライは無い。

#### ② node --watch(nodemon の置換)
- **目的**: 開発時のファイル変更で自動再起動。
- **コード**: `node --watch server.js`
- **Gotcha(検索結果)**: 既定ではエントリファイルと import されたモジュールのみ監視。`fs.readFile()` で読む JSON、テンプレート、`.env` は再起動を起こさない。`--watch-path=./config` で追加できるが macOS と Windows のみ対応。
- **効果**: 設定ゼロで依存0。

#### ③ --env-file(dotenv の置換)
- **目的**: `.env` を読み込み `process.env` に展開。
- **コード(検索結果)**: `node --env-file=.env app.js` — アプリの最初の1行が走る前に値が `process.env` に入る。コード内からは `process.loadEnvFile('.env')`。
- **効果**: ローダのコード/依存が不要。起動前に確定するため import 順序の問題が無い。

#### ④ util.styleText(chalk / kleur の置換)
- **コード(補足例)**:
```js
import { styleText } from 'node:util';
console.log(styleText('red', 'error'));
console.log(styleText(['bold', 'green'], 'ok'));
```
- **効果**: ANSI エスケープを標準で生成。Gotcha: chalk のチェーン API や色レベル自動検出の細かさは無い。

#### ⑤ node:test / node:assert(jest / mocha の置換)
- **コード(補足例)**:
```js
import test from 'node:test';
import assert from 'node:assert/strict';
test('adds', () => { assert.equal(1 + 1, 2); });
```
実行: `node --test`
- **効果(検索結果)**: 多くのプロジェクトで、テスト依存も設定ファイルも無しで完結。Gotcha: jest のスナップショット/モック周辺エコシステムは薄い。

#### ⑥ node:sqlite(better-sqlite3 の置換)
- **コード(補足例)**:
```js
import { DatabaseSync } from 'node:sqlite';
const db = new DatabaseSync(':memory:');
db.exec('CREATE TABLE t(id INTEGER PRIMARY KEY, name TEXT)');
db.prepare('INSERT INTO t(name) VALUES (?)').run('a');
console.log(db.prepare('SELECT * FROM t').all());
```
- **効果**: ネイティブアドオンのビルド不要。Gotcha: Node 24 では RC 段階で API 変更の可能性。

#### ⑦ fs.glob(glob の置換)
- **コード(補足例)**: `import { globSync } from 'node:fs'; const files = globSync('src/**/*.js');`
- **Gotcha**: glob パッケージの全オプションは未対応。

---

## 2. その他のニュース(検索結果より)

1. **Node v22.23.3 (LTS)**: ルート証明書と依存(OpenSSL, npm, Undici)の更新、軽微な修正。
2. **Node.js 27 から Intel Mac は "experimental" 扱い**: 公式 x64 macOS ビルドなし、arm64 のみのインストーラ。x64 Mac ユーザーは影響確認が必要。
3. **`--process-timeout` フラグ提案**: タイトループで固まったプロセスでも強制終了し、何がプロセスを生かしていたかを報告する。ハング診断が目的。
4. **GitHub Actions が Node 20 ランナーを廃止**: カスタムアクションは `runs.using` を `node24` に変更し新リリースを公開する必要がある。
5. **node バイナリを約1/3縮小する PR**: macOS arm64 で 147MB → 95MB。追加圧縮とシンボル除去。作業に Grok を使用した点も話題。
6. **AI 支援実験で Ada / simdjson / simdutf が高速化**: Node 同梱ライブラリの性能改善事例。
7. **npm の新トークン権限**: CI がリリースを「ステージ」できるが公開不可。メンテナが 2FA で各ステージ版を承認。npm は 2027年1月に bypass-2FA トークンによる直接公開を終了予定で、その移行パスになる。
8. その他: pnpm 12.6(ワークスペース重複排除と node_modules 移設対応)、Linear の CI 刷新(AI 生成テストでスイートが4倍化)、シニアエンジニアの燃え尽きに関する記事。

※ 2〜8 はリンク先本文を取得できず、概要レベルのみ。コード・デモは未確認。
