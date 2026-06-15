# Node Weekly Issue 627 (June 4, 2026) まとめ

> 原文: https://nodeweekly.com/issues/627

---

## 1. replacements.fyi — npm パッケージの代替手段を探すツール

**リンク先:** https://replacements.fyi/ / https://e18e.dev/blog/replacements-fyi-launch

### ① 目的・問題意識

Node.js エコシステムには、現代の Node.js ランタイムが既にネイティブでサポートしている機能を提供するだけの npm パッケージが多数存在する。`chalk`（ターミナル色付け）、`axios`（HTTP クライアント）、`is-number`（数値判定）などがその代表例だ。これらの軽量ラッパーが不要なのに使われ続ける理由の一つは、「どれが代替できるか」を調べる手段がなかったことにある。

e18e (Ecosystem Performance) コミュニティが管理する `module-replacements` データセットには、~100 件以上の PR によってドキュメント化されたパッケージ置き換え情報が蓄積されていたが、これを一般開発者が検索できる UI は存在しなかった。`replacements.fyi` はその課題を解決するために生まれた。

### ② 具体的な技術詳細・コード例

**サイトの仕組み:**
- パッケージ名を入力すると、対応する代替手段（ネイティブ API、コードスニペット、または別パッケージ）を表示する
- Node.js ランタイムバージョンでフィルタリング可能（ターゲット環境に合った代替を提示）
- AI エージェントがアクセスしやすいよう agent-friendly 設計

**代替例 1: `chalk` → `node:util` の `styleText`**

```js
// Before: chalk を使った場合
const chalk = require('chalk');
console.log(chalk.red('エラー!'));
console.log(chalk.blue.bold('情報'));
console.log(chalk.bgGreen.white('成功'));

// After: Node.js 組み込み util.styleText を使う
const { styleText } = require('node:util');
console.log(styleText('red', 'エラー!'));
console.log(styleText(['blue', 'bold'], '情報'));
console.log(styleText(['bgGreen', 'white'], '成功'));
```

- `util.styleText` は Node.js v20.12.0 以降で利用可能（v22.13.0 で Stable に昇格）
- 自動移行ツール（codemod）も提供されている: `npx codemod @nodejs/chalk-to-util-styletext`
- 非対応メソッド（`hex()`, `rgb()`, `ansi256()` 等）は警告が出る

**代替例 2: `axios` → ネイティブ `fetch()`**

```js
// Before: axios を使った場合
const axios = require('axios');
const response = await axios.get('https://api.example.com/data');
const data = response.data; // axios が自動で JSON をパース

// After: Node.js 18+ のグローバル fetch を使う
const response = await fetch('https://api.example.com/data');
const data = await response.json(); // 明示的なパース

// POST の場合
// Before:
await axios.post('/api/endpoint', { key: 'value' });

// After:
await fetch('/api/endpoint', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' }),
});
```

- `fetch()` は Node.js 18 からグローバルで利用可能（Node.js 21 以降で完全安定）
- エラーハンドリングの違い：`axios` は 4xx/5xx でエラーを throw するが、`fetch` は `response.ok` を自分で確認する必要あり

**代替例 3: `is-number` → インラインスニペット**

```js
// Before: is-number パッケージ
const isNumber = require('is-number');
isNumber(1); // true
isNumber('1'); // true

// After: ネイティブの一行チェック（各用途に応じて選択）
// 文字列も含め数値とみなす場合
const isNumeric = (n) => !isNaN(parseFloat(n)) && isFinite(n);

// 厳密な数値型チェックの場合
const isStrictNumber = (n) => typeof n === 'number' && !isNaN(n);
```

**代替例 4: dotenv → Node.js 組み込み `--env-file`**

```bash
# Before: dotenv パッケージ + require('dotenv').config()
node app.js

# After: Node.js 20.6+ の組み込み機能
node --env-file=.env app.js
```

```js
// または process.loadEnvFile() をコードから呼ぶ
process.loadEnvFile('.env'); // Node.js 21.7+
```

**e18e ESLint プラグインとの連携:**

```bash
npm install --save-dev @e18e/eslint-plugin
```

```json
// eslint.config.js (flat config)
{
  "plugins": {
    "e18e": require("@e18e/eslint-plugin")
  },
  "rules": {
    "e18e/no-useless-packages": "warn"
  }
}
```

→ 使われているパッケージを静的解析し、不要な依存を警告として表示する。

**npmx との連携:**

npmx (https://npmx.dev) でパッケージ情報を閲覧すると、置き換え可能なパッケージに対してバナーが表示される。例えば `chalk` を検索すると「Node.js の `util.styleText` で代替できます」という通知が表示される。

**開発ストーリー:**

e18e Discord での一つのメッセージから始まり、2日間で本番サイトとして公開された。公開前の1週間だけで 26件の PR がマージされ、コミュニティ主導の開発が急速に進んだ。

### ③ なぜ効果的か

1. **依存削減によるセキュリティ向上**: 不要な npm パッケージはサプライチェーン攻撃のリスク要因となる（2026年6月の Miasma ワームは npm パッケージへの侵害を利用）。ネイティブ API への置き換えによりリスクを排除できる。
2. **パフォーマンス改善**: node_modules のサイズが減り、インストール時間・起動時間・メモリ消費が改善する。
3. **メンテナンスコスト削減**: 外部パッケージのバージョン管理や脆弱性対応が不要になる。
4. **AI エージェント統合**: replacements.fyi が agent-friendly に設計されているため、AI コーディングアシスタントが「不要な依存を追加しないコード」を生成しやすくなる。
5. **コミュニティ主導のデータ品質**: module-replacements データセットは GitHub での PR レビューを経ており、品質が保証されている。

---

## 付録: e18e エコシステム概要

| ツール | 説明 | リンク |
|--------|------|--------|
| `replacements.fyi` | パッケージ代替を検索するウェブサイト | https://replacements.fyi/ |
| `module-replacements` | 代替情報のデータセット（npm パッケージ） | https://npmjs.com/package/module-replacements |
| `@e18e/eslint-plugin` | ESLint で不要な依存を警告するプラグイン | https://github.com/e18e/eslint-plugin |
| `npmx` | module-replacements データを UI 表示するパッケージブラウザ | https://npmx.dev |
| `module-replacements-codemods` | 自動マイグレーション codemod 集（~150件） | GitHub: e18e/module-replacements-codemods |

---

*※ 本号に含まれる "Memetria K/V: Efficient Redis & Valkey Hosting" はスポンサー記事のため省略。*
