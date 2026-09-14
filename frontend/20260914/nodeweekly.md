# Node Weekly Issue #640 まとめ

> **Issue #640: Node is getting a built-in benchmark runner**
> 出典: [https://nodeweekly.com/issues/640](https://nodeweekly.com/issues/640)
> 発行日: 2026年9月上旬（推定。nodeweekly.com への直接アクセスが 403 のため正確な発行日は未確認）

> **注記**: nodeweekly.com および daily.dev への直接アクセスは組織のネットワークポリシーによりブロックされているため、WebSearch ツールで得られたスニペットと、各記事のリンク先（GitHub / npm / GHSA 等、アクセス可能なドメイン）を直接調査して本まとめを作成した。一部の記事（#9, #10）はリンク先の特定ができなかったため、確認できた範囲のみ記載している。

---

## 目次

1. [Node.js に組み込みベンチマークランナー `node:bench` が提案される](#1-nodejs-に組み込みベンチマークランナー-nodebench-が提案される)
2. [vgpu: 同じ WebGPU シェーダーをブラウザとヘッドレス Node で動かすライブラリ](#2-vgpu-同じ-webgpu-シェーダーをブラウザとヘッドレス-node-で動かすライブラリ)
3. [Rslib 1.0 リリース](#3-rslib-10-リリース)
4. [MikroORM 7.2: ネイティブ Row Level Security 対応](#4-mikroorm-72-ネイティブ-row-level-security-対応)
5. [parse-xml 5.0: ESM-only 化](#5-parse-xml-50-esm-only-化)
6. [Fastify 5.12.2: 複数の高深刻度脆弱性を修正](#6-fastify-5122-複数の高深刻度脆弱性を修正)
7. [pnpm 12.4: クロス言語ワークスペース対応](#7-pnpm-124-クロス言語ワークスペース対応)
8. [Nodemailer 10.0: TypeScript 化と ESM/CJS デュアルビルド](#8-nodemailer-100-typescript-化と-esmcjs-デュアルビルド)
9. [dhcp.js 1.0（情報限定）](#9-dhcpjs-10情報限定)
10. [Node.js と VS Code の歴史（情報限定）](#10-nodejs-と-vs-code-の歴史情報限定)

---

## 1. Node.js に組み込みベンチマークランナー `node:bench` が提案される

**原文リンク:** [bench: add experimental `node:bench` module by jasnell · Pull Request #65606](https://github.com/nodejs/node/pull/65606) (nodejs/node)

### ① 目的・問題意識

Node.js のベンチマーク計測には従来サードパーティ製ツール（`bench-node`、`benchr`、`bench-runner` 等）が乱立しており、統一された最小限のプリミティブが存在しなかった。本 PR は `node:test` が果たしてきた役割（＝標準の最小カーネルを提供し、その上に高機能ツールが積み上げられる）を、ベンチマーク領域でも実現することを目的とする。フルスペックのベンチマークツールを置き換えるのではなく、Node.js 自身のベンチマーク基盤としても使え、サードパーティツールの土台にもなる軽量モジュールを目指している。

### ② 具体的なコードと技術詳細

`node:test` と非常によく似た設計が採用されている。

```javascript
import { bench, suite } from 'node:bench';

suite('URL', () => {
  const input = 'https://example.com/a?b=c';

  bench('construct', {
    samples: 30,
    params: { input: 'short' },
  }, (b) => {
    const operations = 10_000;

    b.start();
    for (let i = 0; i < operations; i++) {
      new URL(input);
    }
    b.end(operations);
  });
});
```

実行方法:

```bash
node --bench benchmark.mjs
```

- `suite()` でベンチマーク群をグルーピングし、`bench()` で個々の計測を定義する
- `samples` オプションでサンプル数、`params` でパラメータ化されたベンチマーク（複数条件での比較）を指定できる
- コールバック内で `b.start()` / `b.end(operations)` を呼び出し、計測区間と処理件数を明示する（`operations` を渡すことで ops/sec 等の指標が算出される）

設計上の重要な判断として、作者（jasnell）は既存のベンチマークパッケージをそのまま Node 本体に取り込む（vendoring）ことを明確に拒否し、「人によって欲しいものが違う（different folks actually want different things）」という理由で、あくまで最小限の計測・構造化された結果のプリミティブのみを提供する方針を採った。

### ③ なぜそれが効果的か

- **依存ゼロの最小基盤**であるため保守コストが低く、Node コア自体の内部ベンチマークにも使える
- `node:test` と同じ設計思想を踏襲することで、既存の Node.js 開発者にとって学習コストが低い
- 最小カーネルの上に Vitest のような高機能フレームワークが `node:test` と共存しているのと同様に、`node:bench` の上にも多様なツールが構築できる拡張性を確保している
- 何を「機能として持たないか」を明確にすることで、スコープクリープを防ぎつつ実用的なベースラインを提供している

---

## 2. vgpu: 同じ WebGPU シェーダーをブラウザとヘッドレス Node で動かすライブラリ

**原文リンク:** [vercel-labs/vgpu](https://github.com/vercel-labs/vgpu) / [vgpu.sh](https://vgpu.sh/)（Vercel が2026年8月にオープンソース化）

### ① 目的・問題意識

WebGPU を使ったシェーダー開発は、ブラウザの `<canvas>` 上でのインタラクティブな確認と、CI でのテストや動画書き出しなどのヘッドレス実行とで、これまで別々の実装・別々のライブラリを必要としていた。`vgpu` は「同じ WGSL シェーダーコードを一度書けば、ブラウザのインタラクティブキャンバス、ヘッドレスレンダリング、画像・動画出力、自動テストのすべてで使い回せる」ことを目的としたクロスランタイム設計の TypeScript ライブラリである。AI エージェントによるシェーダー開発支援も念頭に置かれている。

### ② 具体的なコードと技術詳細

**ヘッドレス Node.js での実行例:**

```javascript
import { draw, frame, init, target } from "vgpu/node";
import triangleShader from "./triangle.wgsl";

const gpu = await init();
const colorTarget = target(gpu, { size: [256, 256], format: "rgba8unorm" });
const triangle = draw(gpu, { shader: triangleShader });

frame(gpu, (f) => f.pass(colorTarget, triangle));
const pixels = await colorTarget.read();
gpu.dispose();
```

処理の流れ：
1. `init()` で GPU デバイスを初期化（Node 版は Dawn バックエンド）
2. `target()` でオフスクリーンのレンダーターゲット（サイズ・フォーマット指定）を作成
3. `draw()` で WGSL シェーダーを束ねた描画コマンドを構築
4. `frame()` でフレームを実行し、`pass()` でターゲットに描画を適用
5. `colorTarget.read()` でピクセルデータを取得（画像化・動画化・テスト検証などに利用可能）
6. `gpu.dispose()` でリソースを明示的に解放

**アーキテクチャ上の特徴:**

- **型付き WGSL インポート**: シェーダーファイル自体を TypeScript モジュールとしてインポートでき、バインディングや型情報が自動的にリフレクションされる
- **単一の `Gpu` ハンドル**: すべての操作が第一引数として `Gpu` を受け取るため、隠れたグローバル状態が存在しない
- **軽量**: 「フルスクリーンエフェクト一式が gzip 圧縮後 25KB」に収まることを CI で強制している
- 公開 API は `init` / `draw` / `compute` / `effect` / `frame` / `bundle` / `target` / `uniforms` などで構成され、ブラウザ・ヘッドレス Node (`vgpu/node`, Dawn バックエンド)・決定論的モック (`vgpu/mock`, テスト/CI用) の3環境で同一の API サーフェスを提供する
- **エージェント対応**: `npx vgpu docs` / `npx vgpu examples` / `npx vgpu check` などの CLI がドキュメント参照・サンプル取得・シェーダー検証を提供し、`vgpu.sh` は LLM 向けに `agents.md` と `llms.txt` を公開。読み取り専用の MCP エンドポイント（`https://vgpu.sh/api/mcp`）にも接続できる

### ③ なぜそれが効果的か

- **ブラウザ／ヘッドレス／テストで完全に同じコードが動く**ため、デモ用のインタラクティブ実装とCI用の自動検証実装を二重に書く必要がない
- **明示的なフレーム制御**（暗黙のシーングラフを持たない）により、何がいつ描画されるかが予測可能でデバッグしやすい
- **バンドルサイズが CI でガードされている**ため、パフォーマンス予算が長期的に守られる
- MCP エンドポイントや `llms.txt` によって、AIエージェントがシェーダーコードを生成・検証するワークフローに組み込みやすい

---

## 3. Rslib 1.0 リリース

**原文リンク:** [Announcing Rslib 1.0](https://rslib.rs/blog/v1-0) / [web-infra-dev/rslib](https://github.com/web-infra-dev/rslib)

### ① 目的・問題意識

Rslib は Rspack（Rust製の高速バンドラー）をベースにした「ライブラリをビルドするためのツール」で、JavaScript ライブラリを様々な形式（ESM/CJS/UMD/IIFE 等）でシンプルにビルドすることを目的とする。1.0 では特にビルド速度と TypeScript 宣言ファイル生成速度の大幅な改善、Module Federation 出力対応が主眼となっている。

### ② 具体的な技術詳細

**設定例（最小構成）:**

```javascript
export default {
  lib: [
    { format: 'esm' },
    { format: 'cjs', syntax: 'es2020' }
  ],
  syntax: 'es2023'
};
```

**設定ファイル無しでの実行:**

```bash
npx rslib build --entry src/index.ts --format esm --dts
```

**v0.7 → v1.0 のパフォーマンス改善（React コンポーネント1万個でのベンチマーク）:**

| 指標 | v0.7 | v1.0 | 改善率 |
|------|------|------|--------|
| キャッシュなしビルド | 1.212s | 0.918s | 24.3%高速化 |
| キャッシュありビルド | 1.125s | 0.487s | 56.7%高速化 |
| 出力サイズ（圧縮前） | - | - | 32.2%削減 |
| 出力サイズ（圧縮後） | - | - | 4.1%削減 |

**型宣言ファイル生成速度の比較:**

| 手法 | 所要時間 | 倍率 |
|------|---------|------|
| TypeScript 6（従来） | 9.7秒 | 基準 |
| TypeScript 7 | 4.1秒 | 2.4倍高速 |
| Isolated Declarations | 2.3秒 | 4.2倍高速 |

**ビルドモード:**
- **Bundle モード**: 依存関係を少数のファイルにまとめる（SDK・CLI向け）
- **Bundleless モード**: ソースファイル構造を保ったまま個別にコンパイル（コンポーネントライブラリ向け）

**その他の対応:**
- ESM / CJS / UMD / IIFE に加え、**Module Federation** 出力形式（`format: 'mf'`）に対応し、ランタイムで複数アプリから remote module として読み込み可能
- Node.js の Single Executable Applications (SEA) を使った実験的な実行可能ファイル生成
- React / Vue / Svelte / Solid を Rsbuild プラグイン経由で統合コンパイル
- Rstest（テスト）、Rspress（ドキュメント）、Rsdoctor（ビルド解析）とのエコシステム連携
- AIコーディングアシスタント向けの Agent Skills 対応

### ③ なぜそれが効果的か

- キャッシュ有効時に**56.7%のビルド高速化**は、開発中の反復ビルドサイクルを大きく短縮する
- **TypeScript 7 / Isolated Declarations** による宣言ファイル生成の高速化（最大4.2倍）は、大規模モノレポでの型チェック待ち時間を削減する
- Module Federation 出力に標準対応することで、マイクロフロントエンド構成でのライブラリ共有が設定ファイルレベルで完結する
- 設定ファイルなしでも `npx rslib build` だけで動くため、小規模ライブラリでの導入コストが低い

---

## 4. MikroORM 7.2: ネイティブ Row Level Security 対応

**原文リンク:** [MikroORM 7.2: Trust Issues](https://mikro-orm.io/blog/mikro-orm-7-2-released) / 関連 Discussion: [#6137](https://github.com/mikro-orm/mikro-orm/discussions/6137)

### ① 目的・問題意識

マルチテナントアプリケーションで PostgreSQL の Row Level Security (RLS) を使う場合、従来は「リクエストごとに `SET app.tenant = 'xxx'` のようなセッション変数をコネクション取得時に設定する」という処理を、Knex/Kysely のコネクションプールフック（`onReserveConnection` 等）や `AsyncLocalStorage` を使って**アプリケーション側で自前実装**する必要があった。MikroORM 7.2 はこれをネイティブ機能として ORM に統合し、ポリシーをエンティティのメタデータとして宣言的に管理できるようにした。

### ② 具体的なコードと技術詳細

**移行前（コミュニティで使われていた自前実装パターン、Discussion #6137 より）:**

```javascript
// Kysely の onReserveConnection を使ったセッション変数の設定（従来の回避策）
onReserveConnection: async (connection) => {
  const { userId } = getActionContext();
  const sql = userId && isUuid(userId)
    ? `SET app.current_user_id = '${userId}'`
    : `RESET app.current_user_id`;
  await connection.executeQuery({ sql, parameters: [] });
}
```

**v7.2 でのネイティブ対応（PR #7984 により統合）:**

- RLS ポリシーをエンティティのメタデータとして定義し、スキーマジェネレーターが管理する
- リクエストごとのセッションコンテキストをコネクションにプッシュダウンする仕組みが組み込まれた
- ポリシーは `current_setting('app.tenant')` のような PostgreSQL のセッション変数経由でテナント状態を読み取る
- `em.fork()` にセッション変数を渡してテナントスコープを切り替える:

```javascript
const em = orm.em.fork({
  session: { role: 'app_user', variables: { 'app.tenant': tenantId } }
});
```

**7.2 のその他の主な変更点:**
- to-one リレーションをサブクエリで解決する新しい `through` オプション
- カスタム型・null許容ソートキーに対応したカーソルページネーションの再実装
- ドキュメントに組み込みのライブプレイグラウンドを実現する `sql.js` ドライバー
- `em.execute()` での名前付きパラメータ対応
- 文字列正規化、ネイティブクライアントアクセス
- `await using` サポート（`MikroORM` インスタンスが async disposable プロトコルを実装し、明示的なリソース管理でコネクションが自動クローズされる）
- `@ManyToMany` の結合カラムに `index: true`（または `p.manyToMany(User).index()`）でインデックスを張れるように

### ③ なぜそれが効果的か

- RLS ポリシーが**エンティティメタデータとして宣言的**に管理できるため、SQL の `SET`/`RESET` 文をアプリケーションコードに手書きする必要がなくなり、テナント分離漏れのリスクが下がる
- スキーマジェネレーターがポリシーを管理することで、マイグレーションとポリシー定義が一元化される
- `await using` による async disposable 対応で、コネクションリークを防ぐリソース管理がより宣言的に書ける
- 名前付きパラメータにより生SQL実行時の可読性・保守性が向上する

---

## 5. parse-xml 5.0: ESM-only 化

**原文リンク:** [rgrove/parse-xml](https://github.com/rgrove/parse-xml)

### ① 目的・問題意識

`parse-xml` は Node.js とブラウザ双方で使える、ネイティブコード依存のない軽量XMLパーサーである。標準準拠・わかりやすいエラーメッセージ・依存ゼロを重視して開発されている。5.0 では、エコシステム全体の ESM 移行トレンドに合わせてパッケージを ESM-only にした。

### ② 具体的な技術詳細

- v5.0.0 は **ESM-only** パッケージとなったが、Node.js 22 以降であれば `require()` でもそのまま読み込める（Node.js の `require(esm)` 安定化の恩恵を受ける形）
- API 自体はこのリリースで変更されておらず、対応 Node バージョン上であれば既存コードは無修正で動作する
- パフォーマンス面では、小さいドキュメントに対して 210,841 ops/s を記録しており、`fast-xml-parser` や `xmldoc` など競合ライブラリより高速（大きいファイルではネイティブ実装を使う `libxmljs2` の方が高速）
- ゼロ依存、TypeScript 型定義同梱、XML 1.0 準拠、minified+gzip で約4.2KBという小さなバンドルサイズを維持

### ③ なぜそれが効果的か

- ESM-only 化は、パッケージ側のメンテナンスコスト（CJS/ESMのデュアルビルドの複雑さ）を下げつつ、Node 22+ の `require(esm)` により**利用者側の互換性はほぼ損なわれない**という、移行期において実用的なバランスを取っている
- 依存ゼロ・小さいバンドルサイズにより、サーバーサイド・ブラウザ双方への組み込みコストが低い
- 小規模ドキュメントでの高速性は、設定ファイルやAPIレスポンスなど頻繁にパースされる用途に向いている

---

## 6. Fastify 5.12.2: 複数の高深刻度脆弱性を修正

**原文リンク:** [GHSA-247c-9743-5963 (CVE-2026-33806)](https://github.com/advisories/GHSA-247c-9743-5963) / CVE-2026-76169 (GHSA-p68q-wchp-6fh7) ほか

### ① 目的・問題意識

2026年9月4日ごろに Fastify 5.12.2 が「クラスター化されたセキュリティリリース」として公開され、認証バイパスやバリデーションバイパスにつながる複数の高深刻度（CVSS 7.5 / High）脆弱性がまとめて修正された。主な対象は以下の4件: CVE-2026-76169 (GHSA-p68q-wchp-6fh7)、CVE-2026-84469、CVE-2026-84428、CVE-2026-84504、および CVE-2026-33806 (GHSA-247c-9743-5963)。

### ② 具体的な技術詳細

**CVE-2026-33806: Content-Type ヘッダー先頭スペースによるボディスキーマバリデーションバイパス**

原因は「パーサー」と「バリデーター」で `Content-Type` ヘッダーの正規化処理が異なっていたこと：

- パーサー（`lib/content-type.js`）: `trimStart()` を適用してから処理 → 正しくコンテンツタイプを識別
- バリデーター（`lib/validation.js`）: `trim()` する前に `/[ ;]/` で分割 → 先頭スペースが区切り文字として扱われ空文字列になり、スキーマ検索に失敗してバリデーションがスキップされる

このリグレッションは、CVE-2025-32442 対応のために分割デリミタを `';'` から `/[ ;]/` に変更した 2025年4月のコミット `f3d2bcb` に起因する（旧コードは分割後に `.trim()` していたため問題がなかった）。

脆弱なバージョン: Fastify >= 5.3.2 〜 5.8.4 / 修正版: 5.8.5+

```javascript
// 攻撃例: 先頭にスペースを入れた Content-Type でバリデーションをすり抜ける
const res = await fastify.inject({
  method: 'POST',
  url: '/transfer',
  headers: { 'content-type': ' application/json' }, // 先頭にスペース
  payload: JSON.stringify({ amount: 9999, admin: true })
});
// 本来は 400 のはずが 200 が返ってしまう
```

修正後のロジック（バリデーター側にも `trimStart()` を追加）:

```javascript
function getEssenceMediaType(header) {
  if (!header) return ''
  return header.trimStart().split(/[ ;]/, 1)[0].trim().toLowerCase()
}
```

**CVE-2026-76169: 404ハンドラー経由のプレフィックスカプセル化認証バイパス**

- 影響バージョン: Fastify >= 4.0.0 〜 5.12.2未満
- 不正な形式のURLを、あるプラグインのプレフィックス配下に送ることで、**別の兄弟プラグインの not-found ハンドラー**にルーティングでき、しかも本来そのハンドラーに設定されているはずの `preHandler`（認証フック）を経由せずに呼び出せてしまう
- 内部的には、URLデコード前の共有ハンドラーポインタを介して不正なパスをディスパッチしており、プレフィックスを無視して該当ハンドラーの通常のライフサイクルをスキップしていたことが原因
- 影響: 未認証の攻撃者が、無関係な公開プレフィックス経由で、認証保護されたプライベートなフォールバックハンドラーの完全なレスポンスを読み取れる
- 修正（5.12.2）: `lib/four-oh-four.js` から共有の `_routeEventHandler` ポインタを廃止し、`onBadUrl` / `onMaxParamLength` を `routerOptions` に直接結線。不正なURLはどのアプリケーションレベルの not-found ハンドラーが実行される前に fail-closed するようになった

**CVE-2026-84504: 非同期バリデーターの `{value, error}` アンラップに関する不整合**

- 非同期バリデーターが、同期コンパイラーと同じ方法で解決済みの `{value, error}` オブジェクトをアンラップしなくなったため、ペイロードがそれらのキーを通じてリクエストパートを置き換えることができなくなった（脆弱性の修正内容）

いずれも CVSS 7.5（High）、ネットワークアクセスのみで認証・ユーザー操作は不要。

### ③ なぜそれが効果的か

- パーサーとバリデーターの正規化ロジックを**同一関数に統一**することで、今後同種の差分バグ（parser-validator differential）が再発しにくくなる
- not-found ルーティングの共有ハンドラーポインタを廃止し `routerOptions` に直接結線したことで、**プラグインのプレフィックスによるカプセル化**という Fastify のコア設計原則がURLデコード前の段階から保証されるようになった
- 4件の脆弱性を1つのクラスターリリースとしてまとめて修正・告知することで、利用者は一度のアップグレードで包括的に対応できる

---

## 7. pnpm 12.4: クロス言語ワークスペース対応

**原文リンク:** [pnpm 12.4.0 Release Notes](https://github.com/pnpm/pnpm/releases/tag/v12.4.0)

### ① 目的・問題意識

近年のプロジェクトは npm パッケージだけでなく、Python パッケージ（AI/ML関連ツールなど）や Rust の crate（napi-rs 等によるネイティブアドオン）を同一ワークスペース内で扱うケースが増えている。従来は言語ごとに別々のパッケージマネージャー（pip/uv、cargo）を個別に実行する必要があった。pnpm 12.4 は、これらを pnpm のワークスペースの仕組みに統合することを目的とする。

### ② 具体的な技術詳細

**クロス言語対応:**

- `pnpm-workspace.yaml` で `cargo.enabled` または `python.enabled` を有効化すると、npm・Python・Cargo の依存関係を同一ワークスペースで管理できる
- Python パッケージの追加: `pnpm add pypi:<package>`（`pyproject.toml` と管理された `.venv` を使用）
- Rust crate の追加: `pnpm add crate:<package>`（crates.io またはカスタムレジストリから取得）
- 各エコシステムは自身のセマンティクスを保持する。例えば Cargo の依存関係は `Cargo.toml` / `Cargo.lock` に記録され、`cargo.indexUrl` で指定されたレジストリから解決され、`.cargo/config.toml` を通じて配線された Cargo ディレクトリソースにベンダリングされる
- これらすべてを単一の `pnpm install` コマンドでインストールできる

**pnpm pipeline（新機能）:**

```bash
# フローズン依存関係をインストールし、ワークスペースのタスクを実行
pnpm pipeline [name]

# タスクグラフをプレビュー（実行はしない）
pnpm pipeline --dry-run
```

- タスクは入力（inputs）・出力（outputs）・環境変数・キャッシュ（ログのリプレイ付き）をサポート
- 影響を受けるプロジェクトを選択し、そのタスクグラフを実行し、あるタスクが失敗しても後続タスクの実行を継続できる

**その他の追加機能:**
- プラットフォーム対応拡大: Android (arm64, x64)、FreeBSD (x64)、Linux (ppc64le, s390x, riscv64)
- `trustPolicyExcludePrune`: 信頼済みパッケージ除外リストから未使用バージョンを自動削除
- `pnpm change check`: ワークスペース設定のバージョニングバンドに対してパッケージバージョンを検証（CI向け）
- レジストリメタデータの分離により、あるレジストリのパッケージバージョンが別レジストリのインストールに混入する問題を防止
- `jsr:@scope/pkg` のようなプロトコル接頭辞付きセレクターに対応
- `--prod=false` のような真偽値フラグの明示的指定に対応

### ③ なぜそれが効果的か

- 一つの `pnpm install` で npm / Python / Rust の依存関係を同時に解決できるため、**AIツールチェーン（Python）とネイティブアドオン（Rust/napi-rs）を含むフルスタックプロジェクト**でツール切り替えの手間がなくなる
- 各エコシステムのロックファイル・レジストリ解決の意味論をそのまま尊重する設計のため、既存の Cargo/pip エコシステムとの互換性を壊さない
- `pnpm pipeline` のタスクグラフ実行とキャッシュ機構により、モノレポでのCIビルド時間を短縮できる

---

## 8. Nodemailer 10.0: TypeScript 化と ESM/CJS デュアルビルド

**原文リンク:** Nodemailer 10.0 リリース（Node Weekly #640 経由。プロジェクト本体ページは今回未到達）

### ① 目的・問題意識

広く使われているメール送信ライブラリ Nodemailer が、コードベースを TypeScript に移行し、ESM と CommonJS の両方をネイティブにサポートするデュアルビルド構成へ刷新した。近年多くの主要ライブラリが ESM-only 化へ向かう中、Nodemailer は互換性を優先し、`import` を使う利用者には ESM を、`require` を使う利用者には CJS をそれぞれ提供する形を選んでいる。

### ② 具体的な技術詳細

- Nodemailer 10 は **TypeScript で書き直され、ESM/CJS のデュアルビルド**として配布される
- 実プロジェクトでの移行例として、6.9.9 → 10.0.1 へのアップグレードに合わせて AWS SES 連携を SESv2 SDK に移行するケースが確認できる（既存の SES 連携コードとの同時アップグレードが必要になりうる点に注意）
- 一般的な TypeScript デュアルビルドの手法（CJS/ESM 別ディレクトリへの出力、`package.json` の `exports` フィールドでの出し分け、型定義の両対応）に沿った構成が採用されている

### ③ なぜそれが効果的か

- TypeScript化により型定義が一次情報として保守されるようになり、型定義の陳腐化（`@types/nodemailer` とのズレ）が起きにくくなる
- ESM/CJS 両対応により、`"type": "module"` を採用したプロジェクトと従来の CommonJS プロジェクトの両方から、変換ツールを挟まずに直接利用できる

---

## 9. dhcp.js 1.0（情報限定）

**関連リンク:** Node Weekly #640 の要約による言及のみ。該当パッケージの正確なリポジトリ・npm ページを本調査では特定できなかった（`dhcpjs`、`node-dhcp`、`@network-utils/dhcp` など類似名の既存パッケージは複数存在するが、どれが「dhcp.js 1.0」に該当するか確証を得られていない）。

### ① 目的・問題意識（確認できた範囲）

Node Weekly の要約文によれば、「dhcp.js 1.0 が DHCPv4 ツールを出荷した」とのことで、Node.js から DHCPv4（動的ホスト構成プロトコル v4）のパケット送受信・解析を行うためのライブラリと推測される。

### ② 具体的な技術詳細

正確なパッケージを特定できなかったため、具体的なAPI・コード例の提示は見送る。参考として、同種の既存ライブラリ（例: `dhcpjs`）では以下のようなサーバー起動パターンが一般的である。

```javascript
// 参考: 類似パッケージ dhcpjs のサーバー起動例（"dhcp.js 1.0" 本体のAPIではない可能性がある点に注意）
var dhcpjs = require('dhcpjs');
var server = dhcpjs.createServer();
server.on('message', function (m) { console.log(m); });
server.on('listening', function (address) { console.log('listening on ' + address); });
server.bind(); // ポート67にバインドするため root 権限が必要
```

### ③ なぜそれが効果的か（確認できた範囲）

DHCPv4のようなネットワークプロトコルをNode.jsから直接扱えることで、ネットワーク診断ツールや組み込み機器のプロビジョニングツールをJavaScriptエコシステム内で完結して構築できる、という一般的なメリットが推測されるが、本記事固有の効果は未確認。

---

## 10. Node.js と VS Code の歴史（情報限定）

**関連リンク:** Node Weekly #640 の要約による言及のみ。「Node's role in VS Code's history」に対応する具体的な記事URLを本調査では特定できなかった。

### ① 目的・問題意識（関連する公知の背景）

直接の記事は特定できなかったが、関連する公知の事実として、VS Code の共同開発者 Erich Gamma が過去に語っている内容によれば、VS Code はもともと失敗したブラウザ内エディタ「Monaco」から方向転換して2015年4月にデスクトップ製品として発表された。クロスプラットフォームのデスクトップフレームワークとして Electron（当時の Atom Shell）を採用する判断は開発の後半で行われ、当時 nw.js と比較検討した上で、開発がより活発だった Electron が選ばれた。Electron は Chromium と **Node.js** の上に構築されており、これにより VS Code は標準的な Web API とファイルシステムアクセス等の Node.js API を組み合わせて実装できている。

### ② 具体的な技術詳細

該当記事本文を確認できなかったため、コード例等の提示は見送る。一般に知られている構成としては、VS Code は UIプロセスと拡張機能（Extension Host）を別プロセスの Node.js 環境で実行するアーキテクチャを採り、拡張機能がエディタ本体をクラッシュさせずに独立してNode.js APIを利用できるようにしている。

### ③ なぜそれが効果的か（関連する公知の背景）

Node.jsをExtension Hostとして分離実行することで、サードパーティ拡張機能の障害がエディタ本体の安定性に影響しにくくなる、という設計上の利点が一般に知られている。ただし本記事固有の論旨（Node.jsが具体的にVS Codeの歴史においてどのような役割を果たしたかの詳細）は原文未確認のため、上記は背景情報に留まる。

---

## まとめ

| トピック | 重要度 | アクション |
|---------|--------|-----------|
| `node:bench` 組み込みベンチマークランナー | ★★★ | 実験的モジュールとして動向を注視し、既存ベンチマークツールとの棲み分けを検討する |
| vgpu (WebGPU headless) | ★★☆ | シェーダーのCIテストや動画生成が必要な場面で採用を検討する |
| Rslib 1.0 | ★★☆ | ライブラリ開発でのビルド速度・型生成速度改善のため乗り換えを検討する |
| MikroORM 7.2 (Row Level Security) | ★★★ | マルチテナントPostgreSQL構成では自前RLS実装からの移行を検討する |
| parse-xml 5.0 | ★☆☆ | Node 22+ 環境なら`require(esm)`経由でそのまま追随可能 |
| Fastify 5.12.2 | ★★★ | **今すぐ**アップグレードし、認証バイパス・バリデーションバイパスに対応する |
| pnpm 12.4 | ★★☆ | Python/Rustを含む混成ワークスペースでの一元管理に活用する |
| Nodemailer 10.0 | ★★☆ | SES連携等の既存コードへの影響を確認した上でアップグレードする |
| dhcp.js 1.0 | ☆☆☆ | 詳細未確認。利用検討時は個別に一次情報を要確認 |
| Node.js と VS Code の歴史 | ☆☆☆ | 詳細未確認。背景情報として一般公知の経緯のみ記載 |
