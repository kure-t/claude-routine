# Node Weekly Issue 637 まとめ (August 13, 2026)

> 出典: [Node Weekly Issue #637: Building terminal UIs with HTML and CSS](https://nodeweekly.com/issues/637)
>
> ※ Node Weekly は次号 (#638) から8月27日まで夏季休止に入るとのアナウンスあり。

---

## 1. TermDOM: HTML・CSS・DOM でターミナル UI を構築する

**原文リンク:** [TermDOM 公式サイト](https://termdom.org/) / [GitHub: bikeshaving/termdom](https://github.com/bikeshaving/termdom)

### ① 目的・問題意識

ターミナル UI（TUI）を作る従来のライブラリ（Ink など）は React に依存していたり、独自の描画 API を学ぶ必要があったりする。TermDOM は「Ink の見た目は好きだが React は使いたくない」という開発者のために、**本物の DOM 実装・CSS カスケード・レイアウトエンジンをターミナルに直接ペイントする**というアプローチを取る。これにより、Web フロントエンドで培った HTML/CSS の知識・フレームワーク資産をそのまま TUI 開発に転用できる。DomTerm（ターミナルを DOM の中に埋め込む）とはちょうど逆の発想（DOM をターミナルの中に埋め込む）。

### ② 具体的な技術詳細・コード

**レンダリングパイプライン:**
- ブラウザのレンダリングパイプラインを「ピクセルのグリッド」ではなく「文字セルのグリッド」に対して実装している。
- CSS の長さ単位をグリッドにマッピング：`1px` = 1行、`1ch` = 1列。これにより box model・flexbox・table のレイアウトが「セル単位」で計算される。
- 各フレームで、変更のあった要素のスタイル・レイアウトを再計算 → セルバッファへペイント → 前フレームとの差分（diff）を計算 → 差分だけを ANSI エスケープシーケンスとして stdout に書き出す。
- 入力は逆方向に流れる：ターミナルのエスケープシーケンスをデコードし、キーボード・マウス・ペーストイベントとして DOM ノードにディスパッチする。
- ネイティブ依存や WASM 依存なしの Pure JS。Node・Bun・Deno 上で動作し、`bun build --compile` などでバイナリ化も可能。
- CSS のカスケード・継承を ANSI カラーに変換、flexbox・table のフルレイアウト計算、フォームコントロールのターミナルネイティブなスタイリング、Web Components / Shadow DOM 対応、CJK・絵文字・結合文字対応など幅広い機能を持つ。
- 公式の TodoMVC 実装は、スタイルシートを差し替えるだけでブラウザ版からターミナル版に変換できる。

**インストール:**

```bash
npm install @b9g/termdom
```

**コード例（プログレスバー UI）:**

```javascript
import {TermDOM} from "@b9g/termdom";

const term = new TermDOM();
term.attach();

const {document} = term;
document.body.innerHTML = `
  <style>
    .card { border: 1px solid #5fafff; padding: 0 1ch; width: 36ch; }
    .title { color: #5fafff; font-weight: bold; }
    .done { color: green; }
    .rest { color: #444; }
    .pct { color: #888; }
  </style>
  <div class="card">
    <div class="title">Installing</div>
    <div>
      <span class="done" id="done"></span><span class="rest" id="rest"></span>
      <span class="pct" id="pct"></span>
    </div>
  </div>
`;

let n = 0;
setInterval(() => {
  n = (n + 1) % 101;
  const cells = Math.round(n / 4);
  document.getElementById("done").textContent = "█".repeat(cells);
  document.getElementById("rest").textContent = "░".repeat(25 - cells);
  document.getElementById("pct").textContent = String(n).padStart(3) + "%";
}, 50);
```

このコードは通常の DOM 操作（`innerHTML` 代入、`getElementById`、`textContent` 更新）をそのまま使い、CSS でスタイリングされたプログレスバーをターミナルに描画する。

### ③ なぜそれが効果的か

- Web 開発者が既に持っている CSS の知識（flexbox、box model、カスケード）をそのままターミナル UI に転用できるため学習コストが低い。
- 差分（diff）ベースの描画により、変更のあった部分のみを再描画するため、フレームごとの全体再描画に比べて効率的。
- React 依存の Ink と異なり、Vanilla JS でも任意のフロントエンドフレームワークでも使える柔軟性がある。
- CSS スタイルシートの差し替えだけで見た目を変更できる（TodoMVC の例）ため、ロジックとプレゼンテーションの分離が徹底されている。

---

## 2. Fastify 6.0 が Alpha 版に到達

**原文リンク:** [fastify/fastify Releases (GitHub)](https://github.com/fastify/fastify/releases) / [v6.0.0 Milestone](https://github.com/fastify/fastify/milestone/6)

### ① 目的・問題意識

Fastify v5 のリリースから2年が経過し、v6 は「spring clean（大掃除）」的なメジャーリリースとして位置づけられている。目的は、非推奨 API・型・機能の一掃と、対応 Node.js バージョンの引き上げによる長期メンテナンス性の向上。

### ② 具体的な技術詳細

`v6.0.0-alpha.0`（8月11日リリース）での主な破壊的変更：

- 非推奨だった型を削除：`FastifyPlugin`、`FastifyLoggerInstance`、`FastifyRequestContext`、`FastifyReplyContext`、`ValidationResult`、`ResSerializerReply`
- 内部 HTTP クライアントである `undici` を v8.x にアップグレード
- エラーハンドラのオーバーライドをデフォルトで無効化

`v6.0.0-alpha.1`（8月13日リリース、今号で紹介）での変更：

- `fix`: レスポンスの raw ヘッダーを削除
- `test`: 警告テストに `process-warning` の `spyWarning` を使用するよう変更
- `feat`: `Reply.prototype.mediaType` プロパティを追加
- `fix(content-type)`: 不正な media type に対しては `undefined` を返すよう修正

最低要求 Node.js バージョンは Node 24 に引き上げられる見込み。「まずは初期テストとして試してほしい」というステータス。

### ③ なぜそれが効果的か

- 非推奨 API を完全に削除することで、内部実装をシンプルに保ち、TypeScript の型定義もクリーンになる。
- `undici` v8 系への追従により、HTTP/1.1・HTTP/2 まわりのパフォーマンスと標準準拠が向上する。
- エラーハンドラのオーバーライドをデフォルト無効化することで、意図しないエラーハンドリングの上書きによる本番障害を防ぎやすくなる。

---

## 3. IN BRIEF（短信）

- **Node のセキュリティトリアージ**: Node.js メンテナの Matteo Collina が、週20〜40件届くセキュリティレポートを「AI の大群」からどう捌いているかを解説。AI が自動生成した誤検知の多いレポートの選別プロセスがテーマ。
- **Bun 1.4 のリリースが目前**: 目玉機能は Node.js 互換性の向上。号の校了時点ではまだリリースされていない。
- **`node:domain` が Node 27 でランタイム非推奨になる可能性**: 2015年からドキュメント上は非推奨だったが、今後は実行時に警告が出るようになるかもしれない。まだ使っているプロジェクトは早めに意見表明が必要。
- **Node 24.18.1 → 24.19.0 アップグレードで OOM クラッシュ報告**: 複数チームから out-of-memory クラッシュの報告があり、24.18.1 へのロールバックで解消するとのこと。原因はまだ未確定。
- **Vercel がワンクリック Node.js アップグレード機構を提供開始**: 非推奨 Node バージョンを使っているデプロイ向け。特に Node 20 ベースの新規ビルドは10月1日以降失敗するようになる。

---

## 4. ミドルウェアを2つ追加したら TypeScript の型が壊れた（Inngest ブログ）

**原文リンク:** [Adding a second middleware broke our TypeScript types (Inngest Blog)](https://www.inngest.com/blog/adding-a-second-middleware-broke-our-typescript-types) / 関連 Issue: [inngest-js #534](https://github.com/inngest/inngest-js/issues/534)

### ① 目的・問題意識

Inngest クライアントに **ミドルウェアを2つ**渡すと、`step.run` の戻り値の型が `{}` に潰れてしまう（1つだけなら問題なく推論される）というバグの原因究明記事。実際に発生するエラーは、返されたオブジェクトのプロパティにアクセスしようとすると TypeScript が `Property does not exist on type 'JsonifyObject<{}>'` と報告するというもの。`step.invoke`（あるミドルウェアの `step.run` の結果を呼び出す関数）でも同様に `Jsonify` が自分自身と合成されるため、ミドルウェア0個でも二重適用により型崩壊が発生する。

### ② 具体的な技術詳細・コード

原因は `Jsonify` 型ユーティリティにあった。`Jsonify` は意図的に `unknown` を広げず保持する設計だが、`unknown` は `JsonValue` に代入不可能なため、`unknown` を含む型はガードをすり抜けて型全体が `{}` に潰れてしまう。

根本原因は `type-fest` の `FilterJsonableKeys` 型で、兄弟型である `FilterDefinedKeys` や `FilterOptionalKeys` とは異なり、結果を `Exclude<…, undefined>` でラップしていなかったこと。

さらに関連する問題として、ユーザーの `tsconfig.json` で `strictNullChecks: false` が設定されていると、`Jsonify` 型が一部プロパティをオプショナル化してしまう不具合も判明した（Issue #534）：

```typescript
const obj = await step.run("get-obj", () => ({
  foo: "bar",
}));
// 期待される型: { foo: string }
// 実際の型:     { foo?: string }
```

この不具合は PR #512（v3.15.5 でリリース）で導入された `UndefinedToOptional` ユーティリティ型が `strictNullChecks: false` と相互作用して起きていた。

**修正方針:** `Exclude` で事後的に `undefined` を取り除くのではなく、キーの**フィルタリング段階で** `undefined` を除外するように変更。この修正は upstream の `type-fest` にも同じ内容で提出された。

### ③ なぜそれが効果的か

- 型のフィルタリングを「事後のクリーンアップ」ではなく「フィルタリングの時点」で正しく行うことで、`unknown` を含む複雑なオブジェクト型でもガードがすり抜けない。
- ミドルウェアを合成しても型推論が破綻しなくなり、Inngest ユーザーが複数ミドルウェアを安心して組み合わせられるようになる。
- upstream の `type-fest` に修正を還元することで、同種のバグに悩む他のライブラリ利用者にも恩恵が及ぶ。

---

## 5. 5つの便利な npx ヘルパー（Jens Oliver Meiert）

**原文リンク:** [5 Useful npx Helpers (meiert.com)](https://meiert.com/blog/5-npx-helpers/)

### ① 目的・問題意識

`npx` を使えばインストール不要でワンオフのツールをすぐ実行できる。著者が日常のフロントエンド作業（HTML の検証・最適化・リンク切れチェックなど）でよく使う5つの `npx` ツールを紹介する記事。

### ② 具体的な技術詳細・コマンド

- **`npx hihtml`** — HTML の妥当性検証（HTML-validate 使用）、非推奨マークアップのチェック（内部で ObsoHTML を使用）、内蔵のリンクチェッカー、HTML Minifier Next によるページ圧縮を1つにまとめた「スーパーツール」。フォルダ全体を対象に `npx hihtml` を実行するだけで一括チェックできる。
- **`npx obsohtml`** — 非推奨・廃止された HTML 要素/属性のチェックに特化。`--all`/`-a` フラグを付けても圧縮処理はブロックしない（あくまで情報提供目的）。
- **`npx css-dedup`** — CSS の宣言（declaration）の重複を解析し、最適化できる箇所を指摘する。
- **`npx htaccess-punk`** — `.htaccess` ファイル内のリダイレクト設定をチェックする。カレントディレクトリ（または指定ディレクトリ）を走査し、`--errors`/`-e` フラグでエラーのみ表示可能。
- **`npx npm-check-updates`** — `package.json` の依存パッケージが最新か手早くチェックする定番ツールとしても言及。

### ③ なぜそれが効果的か

- `npx` によりグローバルインストールなしでツールを都度実行できるため、CI やワンオフのメンテナンス作業に組み込みやすい。
- HTML/CSS の品質チェック（妥当性・非推奨機能・重複・リンク切れ・リダイレクト）を横断的にカバーしており、静的サイトやレガシーページのメンテナンスコストを下げられる。

---

## 6. Node.js における Dual-Write 問題を Outbox パターンで解決する（freeCodeCamp / Gabor Koos）

**原文リンク:** [How to Fix the Dual-Write Problem in Node.js with the Outbox Pattern (freeCodeCamp)](https://www.freecodecamp.org/news/how-to-fix-the-dual-write-problem-in-node-js-with-the-outbox-pattern/)

### ① 目的・問題意識

「データベースへの書き込み」と「メッセージキューへの publish」という**2つの独立した書き込み**を1つの業務トランザクションとして扱おうとすると、片方だけが成功する（Dual-Write 問題）リスクが生じる。例えば、注文をデータベースに保存した直後にアプリがクラッシュすると、SQS へのメッセージ送信が行われず、後続の fulfillment サービスに注文が伝播しない、といった不整合が起こる。この記事は PostgreSQL（注文サービスの DB）・SQS（キュー）・DynamoDB（fulfillment サービスの DB）という具体的な構成で Transactional Outbox パターンをゼロから実装する。

### ② 具体的な技術詳細

- `sqs.send` を直接呼ぶ代わりに、**業務レコードと同じデータベーストランザクション内で** outbox テーブルに1行 INSERT する。
- 別プロセス（relay/poller）が outbox テーブルをポーリングし、`status='pending'` の行を読み取って SQS へ publish する。
- **トランザクション安全性:** トランザクションがロールバックされれば outbox の行も一緒に消えるため、キューに「送信されたはずなのに実体がない」孤立メッセージが発生しない。
- **クラッシュ耐性:** コミット後・relay 実行前にアプリがクラッシュしても、outbox の行は `status='pending'` のまま DB に残っているため、relay が次回のポーリングで確実に拾い上げて publish できる。
- publish 成功後は outbox 行の `status` を `published`（または削除）に更新し、二重処理を防ぐ。

### ③ なぜそれが効果的か

- DB への書き込みと「送信予定」の記録を**同一トランザクション**にまとめることで、"書き込みは成功したがメッセージは飛ばなかった"／"メッセージは飛んだが書き込みは失敗した" という中間状態を原理的に排除できる。
- Relay プロセスが失敗しても、pending 状態の行がある限り再試行できるため、at-least-once 配送を保証しやすい。
- 2つの異なるデータストア（PostgreSQL と DynamoDB）をまたぐ分散システムでも、各サービスがそれぞれ自分のトランザクション境界内で outbox を扱えばよく、分散トランザクション（2相コミット）を避けられる。

---

## 7. Hono の内部実装を深掘りする（Flavio Copes）

**原文リンク:** [A Deep Dive into How Hono Is Built (flaviocopes.com)](https://flaviocopes.com/how-hono-is-built/)

### ① 目的・問題意識

Hono がどうやって1つの HTTP リクエストをルーティング → ミドルウェア → レスポンス生成まで処理しているかを、実装の内部（ルーター選択、Context 生成、ミドルウェア合成、ランタイムアダプタ）に踏み込んで解説する記事。

### ② 具体的な技術詳細

- **SmartRouter による遅延ルーター選択:** `new Hono()` した際、デフォルトでは `SmartRouter` が使われる。まず `RegExpRouter`（全ルートを1つの正規表現にコンパイルする高速ルーター）を試み、登録されたパターンがそれに適合しない場合のみ、最初のリクエスト時に `TrieRouter`（トライ木ベースの柔軟なルーター）にフォールバックする。
- 一度あるルーターが「全ルートテーブルを受理できる」と判定されれば、以降のリクエストではその選択が固定され、選択処理自体はスキップされる（＝初回リクエストのみコストがかかり、以降は高速なルーターがそのまま使われ続ける）。
- **遅延パラメータ抽出:** ルーターのマッチ結果にはハンドラと抽出されたパスパラメータが含まれ、それが `HonoRequest` に渡されてパラメータ抽出に使われる。パラメータの実際の解析は必要になった時点まで遅延される。
- **Context オブジェクト:** リクエストごとに `Context` オブジェクトが生成され、リクエスト・レスポンス・環境変数などをハンドラ間で受け渡すハブとして機能する。
- **オニオン型（onion-style）ミドルウェア合成:** ミドルウェアが `await next()` を呼ぶことで、リクエスト処理が「外側から内側へ」進み、レスポンス処理は「内側から外側へ」戻ってくる、典型的なオニオンモデルでミドルウェアチェーンを構成する。
- **レスポンスヘルパー:** `c.text()` のような糖衣構文は、最終的に標準の Web `Response` オブジェクトにマッピングされる。
- **ランタイムアダプタ:** Node.js・Cloudflare Workers・Bun・Deno といった実行環境ごとの差異は、境界（アダプタ層）で吸収され、コアのアプリケーションロジックには一切触れない。

### ③ なぜそれが効果的か

- `RegExpRouter` を主軸にすることで、ルート数が多い場合でも O(1) に近いマッチング速度を実現しつつ、複雑なパターン（ワイルドカードなど）は `TrieRouter` で正しくフォールバックできる、速度と柔軟性の両立を実現している。
- ルーター選択を初回リクエスト時のみ行い以降は固定するキャッシュ的な設計により、実行時オーバーヘッドを最小化している。
- Web 標準の `Request`/`Response` を土台にしているため、ランタイムアダプタ層を薄く保てて、Node.js だけでなく Cloudflare Workers・Bun・Deno など複数ランタイムへの移植性が高い。

---

## 8. DeepSeek Harness: DeepSeek 発、Node 駆動の新エージェントハーネス

**原文リンク:** [GitHub: deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)

### ① 目的・問題意識

DeepSeek が公開した Claude Code 的なコーディングエージェントハーネス。すでに GitHub スター3万を突破。他の CLI 型ハーネスと異なり、Web UI 経由で動作する点が特徴。「あらゆる機能をプラグインとして扱う」という設計思想を徹底しており、既存の Node プラグインシステム Cordis（その作者を DeepSeek が採用したとのこと）の上に構築されている。

### ② 具体的な技術詳細

- アーキテクチャ全体が Cordis という「時空間コンポーザビリティ（spatiotemporal composability）」という論文由来の概念に基づくプラグインフレームワークの上に構築されている。
- ツール・LLM アダプタ・ファイルアクセス・エージェントループ自体まで、あらゆる機能が共有コンテキストにマウントされる「プラグイン」として実装されている。

**クイックスタート（npm 経由）:**

```bash
npx @deepseek-ai/dsh web
```

デフォルトで `http://127.0.0.1:3080` に Web UI が立ち上がる。

**ソースからの実行:**

```bash
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

- 現在は developer preview 段階であり、「互換性を破壊する変更が今後も発生する」ことが明記されている。
- プラグインリポジトリには `dsh-plugin` トピックを付与することで、コミュニティでの発見性を高める仕組みがある。
- ライセンスは MIT（サードパーティ依存関係は別途開示）。

### ③ なぜそれが効果的か

- 「エージェントループ自体すらプラグイン」という徹底したプラグイン化により、コア部分を交換・拡張しやすく、新しい LLM やツールへの対応を素早く追加できる。
- Cordis という既存の実績あるプラグインランタイムに乗ることで、車輪の再発明を避けつつ、プラグインエコシステムを他の Cordis ベースプロジェクトと共有できる可能性がある。
- Web UI ベースであることで、CLI に不慣れなユーザーやチームでの共有・デモがしやすくなる。

---

## 9. unbash: 高速・ゼロ依存の Bash パーサー

**原文リンク:** [GitHub: webpro-nl/unbash](https://github.com/webpro-nl/unbash)

### ① 目的・問題意識

依存パッケージの未使用検出ツール Knip が、シェルスクリプトを解析するために内部で使っているパーサーを、単独ライブラリとして切り出したもの。入力が Bash のソースコード（コピペされたコマンド・スクリプト全体・他フォーマットに埋め込まれたシェルコード）であるとき、**実行せずに構造を検査したい**というニーズに応える。分類（コマンド・リダイレクト・置換のセキュリティ監査向け分類）、CI 設定ファイル中の実行ファイル/ファイル参照の棚卸し、コマンド呼び出しのリライト、パイプラインや条件式の可視化・構造化プレビュー、コマンド呼び出しの精密な検索・移行、といった用途を想定。

### ② 具体的な技術詳細・コード

- 対応する Bash 構文はコマンド・制御フロー・パイプライン・リダイレクト・代入・複合文・パラメータ/単語展開・プロセス置換・入れ子の置換・coproc・heredoc・herestring・入れ子/生成構文など広範。
- 不正な形式の入力に対しても、ソース位置付きのエラー情報とともに best-effort な部分 AST を返す。

**基本的な使い方:**

```typescript
import { parse } from "unbash";

const ast = parse('if [ -f "$1" ]; then cat "$1"; fi');
```

**AST の出力例:**

```javascript
{
  type: "Script",
  commands: [{
    type: "Statement",
    command: {
      type: "If",
      clause: { type: "CompoundList", commands: [ /* [ -f "$1" ] */ ] },
      then: { type: "CompoundList", commands: [ /* cat "$1" */ ] }
    }
  }]
}
```

**Word（単語）の部分要素へのアクセス:**

```typescript
const word = parse("echo a$(id)b").commands[0].command.suffix[0];
word.parts; // [Literal, CommandExpansion, Literal]
```

**AST からソースへの復元（Printer）:**

```typescript
import { parse } from "unbash";
import { print } from "unbash/printer";

const ast = parse('if [ -f "$1" ]; then cat "$1"; fi');
const script = print(ast);
```

- バンドルサイズ: minify 後 80KB、gzip 後 19KB。依存パッケージはゼロ。
- パフォーマンスは tree-sitter-bash など他の代替ライブラリより 7〜17倍高速。
- ライセンス: ISC。

### ③ なぜそれが効果的か

- ゼロ依存かつ軽量なため、既存ツール（Knip のような静的解析ツール）へ組み込む際のバンドルサイズ・サプライチェーンリスクへの影響が小さい。
- 型付き・ソース位置付きの AST を返すため、単なる文字列パースでは困難な「特定のコマンド呼び出しだけを安全に書き換える」といった精密な操作が可能になる。
- パフォーマンスが既存の代替（tree-sitter-bash 等）より大幅に高いため、CI のようなシェルスクリプトを大量に解析する用途でも実用的な速度が出せる。

---

## 10. njsscan 1.0: セマンティック解析対応の Node.js 向け SAST ツール

**原文リンク:** [GitHub: ajinabraham/njsscan](https://github.com/ajinabraham/njsscan)

### ① 目的・問題意識

Node.js アプリケーションに潜む安全でないコードパターン（XSS、SSRF、`eval` の危険な使用、JWT の不備、NoSQL インジェクションなど）を、`pip install` するだけで導入できるゼロコンフィグの静的解析（SAST）ツールとして検出する。

### ② 具体的な技術詳細

- **2種類の検出手法を併用:**
  1. `libsast` によるシンプルなパターンマッチング
  2. `semgrep` を使ったセマンティック（構文を理解した上での）コードパターン検索 — 約70の semgrep ルールで XSS・SSRF・`eval`・JWT・NoSQL インジェクションなどを検出
- **インストール:** Python 3.10+ が必要（Mac / Linux 対応）

```bash
pip install njsscan
```

- **基本的な使い方:**

```bash
njsscan [path]
```

- **主なオプション:**
  - `--json` / `--sarif` / `--sonarqube` / `--defectdojo` / `--gitlab-sast` / `--html` — 出力フォーマット指定
  - `-o OUTPUT` — 結果をファイルに保存
  - `-c CONFIG` — カスタム設定ファイルの使用
  - `--missing-controls` — セキュリティコントロールの充足確認を有効化
  - `-w` — 警告があった場合に非ゼロの終了コードを返す
  - `-v` — バージョン表示
- **インラインでの抑制:** 特定の検出結果はコード内コメントで抑制可能

```javascript
// njsscan-ignore: rule_id1, rule_id2
```

- **設定ファイル `.njsscan`:** 対象拡張子、除外パス/ファイル、ルール例外、severity フィルタ、ルールごとの severity 上書きなどをカスタマイズ可能。
- **CI/CD 連携:** GitHub Actions、GitHub Code Scanning（SARIF 形式）、GitLab CI/CD、Travis CI、CircleCI などに対応（設定ファイルまたは Docker コンテナ経由）。
- 検出結果は OWASP カテゴリ・CWE 番号・severity・該当ファイル位置・マッチしたコードスニペットを含む構造化レポートとして出力される。

### ③ なぜそれが効果的か

- 単純な正規表現的パターンマッチングだけでなく `semgrep` によるセマンティック解析を組み合わせることで、変数名の書き換えなど表面的な変更では回避できない、より本質的な脆弱性パターンの検出が可能になる。
- ゼロコンフィグかつ `pip install` のみで導入できるため、既存の Node.js プロジェクトに追加のセットアップコストなく組み込める。
- SARIF や GitLab SAST など主要な CI/CD プラットフォームの標準フォーマットに対応しているため、既存のセキュリティダッシュボードにそのまま統合できる。

---

## 11. Hucre 1.0: ゼロ依存のスプレッドシートエンジン

**原文リンク:** [GitHub: productdevbook/hucre](https://github.com/productdevbook/hucre) / [Playground](https://hucre.dev/playground)

### ① 目的・問題意識

XLSX・CSV・ODS などのスプレッドシート形式を、外部依存なし・Pure TypeScript で読み書きするための軽量ライブラリ。SheetJS（gzip 後約300KB）や ExcelJS（同約500KB）といった既存の代替ライブラリに比べ、大幅に軽量なライブラリを目指して開発され、今号のタイミングで安定版 v1.0.0 に到達した。

### ② 具体的な技術詳細・コード

**対応フォーマット:**
- 読み書き両対応: XLSX、ODS、CSV、TSV、JSON、NDJSON、XML、HTML、Markdown
- 読み込みのみ: XLS（Excel 97-2003）、XLSB（バイナリ Excel）

**インストール:**

```bash
npm install hucre
```

**基本的な読み込み:**

```typescript
import { readXlsx, writeXlsx } from "hucre"

const workbook = await readXlsx(buffer)
console.log(workbook.sheets[0].rows)
```

**基本的な書き込み:**

```typescript
const xlsx = await writeXlsx({
  sheets: [{
    name: "Products",
    columns: [
      { header: "Name", key: "name", width: 25 },
      { header: "Price", key: "price", width: 12, numFmt: "$#,##0.00" },
      { header: "Stock", key: "stock", width: 10 }
    ],
    data: [
      { name: "Widget", price: 9.99, stock: 142 },
      { name: "Gadget", price: 24.5, stock: 87 }
    ]
  }]
})
```

**Tree Shaking（必要な機能だけをインポート）:**

```typescript
import { readXlsx, writeXlsx } from "hucre/xlsx"
import { parseCsv, writeCsv } from "hucre/csv"
import { readOds, writeOds } from "hucre/ods"
import { parseJson, writeNdjson } from "hucre/json"
```

**ストリーミング読み込み:**

```typescript
import { streamXlsxRows } from "hucre/xlsx"

for await (const row of streamXlsxRows(buffer)) {
  console.log(row.index, row.values)
}
```

**ストリーミング書き込み（500万行の例）:**

```typescript
function* rows() {
  for (let i = 0; i < 5_000_000; i++) yield [i + 1, Math.random()]
}

return new Response(
  writeXlsxStream(rows(), {
    name: "BigData",
    columns: [{ header: "ID" }, { header: "Value" }],
    freezePane: { rows: 1 }
  })
)
```

**その他の機能:** セルスタイル・フォント・数値フォーマット・塗りつぶし・罫線・配置などのスタイリング、ドロップダウンリスト/数値範囲/日付制約などのデータバリデーション、外部 URL・シート内参照のハイパーリンク、PNG/JPEG/GIF/SVG/WebP 画像（alt テキスト付き）、棒・列・折れ線・円・ドーナツ・散布図・面グラフなどのチャート、共有/配列/動的数式のサポート、データバーやカラースケールを含む13種類の条件付き書式ルール、ピボットテーブルの読み書き、ECMA-376 Agile 暗号化によるパスワード保護（Excel 2010+）、HTML/Markdown へのエクスポート。

- CSV のみをインポートした場合の gzip サイズは約3.7KB と、機能単位で細かくツリーシェイクされる設計。

### ③ なぜそれが効果的か

- ゼロ依存かつネイティブ ESM 対応であるため、SheetJS や ExcelJS に比べてバンドルサイズを大幅に削減でき、フロントエンドやエッジ環境でも扱いやすい。
- 読み込み・書き込みともにストリーミング API を提供しており、500万行規模のデータでも一定メモリで処理できるため、サーバーサイドでの大規模エクスポート処理に向く。
- フォーマットごとにサブパスエクスポート（`hucre/xlsx`、`hucre/csv` 等）が分かれているため、必要な機能だけを読み込むツリーシェイクが効きやすく、実際のバンドルサイズをさらに絞り込める。

---

## 12. リリース情報（その他）

- **NodeBB 4.15.0** — Node 製フォーラムソフトウェア。多数のバグ修正と fediverse（連合型 SNS）対応の強化。
- **smol-toml 1.8** — 小型・高速・TOML 1.1 仕様準拠のパーサー/シリアライザ。
- **BullMQ 6.1** — Redis または Postgres をバックエンドとする高速・信頼性の高い分散キューライブラリ。
- **swift-node 0.1.2** — Node モジュールのネイティブ部分を Swift で書けるようにするツール。
- **Vitest 5.0 Release Candidate** — テストフレームワーク Vitest の次期メジャーバージョンの RC 版。

---

*除外項目: 本号中の "sponsor" 表記記事（Master.dev「Harness Engineering: How to Wrangle Your Agents」、ParadeDB「Full-Text Search in Drizzle Without a Second System」）はスポンサー記事のため対象外としました。*
