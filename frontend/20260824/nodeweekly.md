# Node Weekly まとめ

- 対象号: **Issue #637** — "Building terminal UIs with HTML and CSS" (2026年8月13日発行)
- 出典: https://nodeweekly.com/issues/637
- 取得方法: nodeweekly.com への直接アクセスは403となるため、WebSearch / RSSフィード（cprss.s3.amazonaws.com/nodeweekly.com.xml）経由で最新号（#637）を特定し、記事本文は各リンク先（GitHub, ブログ等）を個別に調査。
- 注記: sponsor表記のある記事（"Harness Engineering: How to Wrangle Your Agents"／master.dev、"Full-Text Search in Drizzle Without a Second System"／ParadeDB）は対象外としています。

---

## 1. TermDOM: Build Terminal UIs with HTML, CSS and the DOM

- リンク: https://termdom.org/ / https://github.com/bikeshaving/termdom
- 著者: Brian Kim

### ① 目的・問題意識

ターミナルUI（TUI）やインタラクティブなCLIを作る際、既存のTUIライブラリ（Inkなど）はReact依存だったり、専用の学習コストが必要だったりする。TermDOMは「Webの技術（HTML/CSS/DOM）をそのままターミナルに持ち込む」ことで、この学習コストを解消しようとする技術。ネイティブ依存やWASM依存を持たないピュアJSで、Node・Bun・Deno上で動作する。

### ② 具体的なコード・技術詳細

インストールと基本例:

```javascript
npm install @b9g/termdom

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

技術詳細（レンダリングパイプライン）:
- ブラウザのレンダリングパイプラインを「ピクセル」ではなく「文字セル（cell）」単位で再実装している。CSS上の `1px` と `1ch` は共に「1セル」として扱われる。
- 変更のあった要素についてスタイル・レイアウトを再計算 → セルバッファへペイント → 前フレームとdiff → 差分をANSIエスケープシーケンスとしてstdoutへ書き込む、という流れで再描画する。
- stdin側のエスケープシーケンスをデコードし、キーボード・マウス・ペーストイベントに変換する。
- CJK・絵文字・結合文字の幅計算、ヘブライ語・アラビア語の文脈依存シェイピングにも対応。
- `querySelector()`、`MutationObserver`、`ResizeObserver`、`getBoundingClientRect()` などの標準DOM APIがセル座標ベースでそのまま使える。
- flexbox・テーブルレイアウト・Web Components（Shadow DOM含む）・フォーム要素（`<input>`, `<textarea>`, `<select>`, ラジオ/チェックボックス）・テキスト選択・readlineキーバインド・代替スクリーンによるフルスクリーンモードにも対応。
- 公式のTodoMVC実装は、ブラウザ版のスタイルシートを差し替えるだけでターミナル上でも動作する。

### ③ なぜ効果的か

- 本物のDOM/CSSOM実装であるため、Web開発の知識・既存のフロントエンドフレームワーク（React/Vue/Svelte等）やブラウザ向けライブラリをほぼそのままターミナルへ転用できる。
- `MutationObserver`ベースの自動再描画により、Web開発と同様の宣言的なUI更新体験が得られる（手動で再描画ロジックを書く必要がない）。
- Node/Bun/Deno全対応かつバイナリコンパイルも可能なため、配布のしやすさとクロスランタイム性を両立している。

---

## 2. Fastify 6.0 Enters Alpha

- リンク: https://github.com/fastify/fastify/releases
- 著者: The Fastify Team

### ① 目的・問題意識

Fastify v5リリースから約2年が経過し、v6は「spring clean（大掃除）」的リリースと位置づけられている。非推奨APIの整理、最小要求Node.jsバージョンの引き上げなど、蓄積した技術的負債を解消することが目的。

### ② 具体的なコード・技術詳細

v6.0.0-alphaで確認されている主な変更点（GitHubリリースノートより）:
- 非推奨だった `FastifyPlugin` 型の削除
- 最小要求ランタイムバージョンの引き上げ（Node 24以上が必須に）
- 非推奨の `FastifyLoggerInstance` を削除し、`FastifyBaseLogger` に一本化
- `Promise.withResolvers` のponyfillを削除（ネイティブ実装前提に）
- 非推奨の `FastifyRequestContext` / `FastifyReplyContext` 型を削除
- エラーハンドラのオーバーライドをデフォルトで無効化
- 非推奨の `ValidationResult` 型エイリアスを削除
- v6.0.0-alpha.2ではセキュリティ修正（GHSA-w2qp-rph6-63g4、GHSA-3m5p-2c4r-xxw2）や `reply.mediaType` の不具合修正も含まれる

### ③ なぜ効果的か

非推奨だったAPI・型を思い切って削除することで内部実装をシンプルに保ち、最新のNode.js機能（ネイティブ`Promise.withResolvers`等）に前提を揃えることで、パフォーマンスや保守性を向上させる。最小要求バージョンを引き上げることで、古いランタイム向けのポリフィル・分岐コードを削減できる。

---

## 3. Adding a Second Middleware Broke Our TypeScript Types（Inngest）

- リンク: https://www.inngest.com/blog/adding-a-second-middleware-broke-our-typescript-types
- 著者: Linell Bonnette (Inngest)

### ① 目的・問題意識

Inngest TypeScript SDKで、ミドルウェアを2つ以上合成（compose）すると、戻り値の型が意図せず `{}`（空オブジェクト型）に潰れてしまうというバグの調査記。原因は「optionalなキーをフィルタする」という一見安全に見えるイディオムが、`undefined` を値のユニオンに引きずり込んでしまうことだった。

### ② 具体的なコード・技術詳細

問題の構造:
- Inngestの `Jsonify` 型は、`unknown` を意図的に（安全側に倒して）そのまま保持し、広げない設計になっている。しかし `unknown` は `JsonValue` に代入不可能なため、`unknown` を含む型は丸ごとガードをすり抜けて `{}` に潰れてしまう。
- キーをフィルタする際、`type-fest` の `FilterDefinedKeys` / `FilterOptionalKeys` は結果を `Exclude<…, undefined>` でラップしているが、`FilterJsonableKeys` はそのラップをしていなかった、という非対称性が根本原因。
- 過去に同種のバグを踏んだ開発者が2箇所は修正したが、3箇所目（`FilterJsonableKeys`）を見落としていた。

最初に試みた（不十分な）修正:
- キーのユニオンを作り、`Exclude` で `undefined` を除去してから `Pick` に渡すアプローチ。しかしこの `undefined` は、そもそも「optionalなプロパティを値のユニオンとして読み取る」ことで混入したものであり、対症療法にしかならない。

根本修正（アップストリームに提出）:
- 値のユニオンを作ってから `undefined` を除去するのではなく、**キーのリマッピング（`as` 構文によるkey remapping）** を使い、各キーを `NotJsonable` に対してテストし、条件に合わないキーをその場で `never` にマッピングして落とす、という手法に変更。
- これにより `undefined` はそもそも値ユニオンのステップを経由しないため、オブジェクトのキー集合に混入しない。結果として最初の適用で正しい型が得られ、`Jsonify` を再適用しても冪等（no-op）になり、すべての合成ポイントが一括で修正される。

### ③ なぜそれが効果的か

「値レベルでゴミ（`undefined`）を除去してから型を組み立てる」のではなく「キーレベルでフィルタし、該当しないキーを`never`に落とす」ことで、`undefined`が型システムに侵入する経路そのものを断つ。これにより1箇所の型ユーティリティ修正で、ミドルウェアの合成回数に依存せず全ての利用箇所が正しく型付けされるようになる（対症療法ではなく根本治療）。

---

## 4. 5 Useful npx Helpers（Jens Oliver Meiert）

- リンク: https://meiert.com/blog/5-npx-helpers/

### ① 目的・問題意識

`npx` を使えば、グローバルインストールせずに一時的なツールをその場で実行できる。日常のJSプロジェクトのメンテナンス（スペルチェック・リンク切れ検出・依存更新・デッドコード検出・画像圧縮）に便利な5つのツールを紹介する記事。

### ② 具体的なコード・技術詳細

1. **cspell**（スペルチェック）
   ```bash
   npx cspell **/*.md
   ```
   設定ファイルで既知の固有名詞などをホワイトリスト化するメンテナンスが必要。

2. **markdown-link-check**（Markdown内リンク切れチェック）
   ```bash
   npx --node-options='--no-deprecation' markdown-link-check -c ~/markdown-link-check.json
   ```
   設定ファイルと併用するのが推奨。

3. **npm-check-updates**（依存パッケージの更新チェック）
   ```bash
   npx npm-check-updates
   ```
   `package.json` のあるフォルダで実行。

4. **Knip**（未使用ファイル・未使用exportの検出）
   ```bash
   npx knip
   ```
   npm-check-updates同様、依存関係だけでなくファイル・export単位まで対象を広げたチェック。ファイル/export検出には誤検知（false positive）が一定数出るとのこと。

5. **image-guard**（ニアロスレスな画像圧縮の自動化）
   ```bash
   npx image-guard
   ```
   対象フォルダで実行し、リポジトリ内画像を自動圧縮する。

### ③ なぜそれが効果的か

いずれもインストール不要で `npx` 経由で即実行できるため、CIやローカルでの一時利用に向く。エイリアスを設定しておけばさらに使い勝手が向上する、と筆者は述べている。継続的な品質チェック（スペル・リンク・依存・デッドコード・画像サイズ）を、専用ツールの導入コストなしに習慣化できる点が有用。

---

## 5. Fixing the Dual-Write Problem in Node.js with the Outbox Pattern（Gabor Koos / freeCodeCamp）

- リンク: https://www.freecodecamp.org/news/how-to-fix-the-dual-write-problem-in-node-js-with-the-outbox-pattern/

### ① 目的・問題意識

「データベースへの書き込み」と「メッセージキューへのpublish」を別々に行う（dual write）と、片方が成功しもう片方が失敗した場合にデータとイベントの不整合が生じる。この「dual-writeの問題」を、分散トランザクションを使わずに解決する定番手法である **Transactional Outbox パターン** を、Node.js・PostgreSQL（注文サービス）・SQS（キュー）・DynamoDB（フルフィルメントサービス側DB）を用いてゼロから実装する記事。

### ② 具体的なコード・技術詳細

- アプリケーションは `sqs.send` を直接呼ぶ代わりに、ビジネスレコードと**同一のDBトランザクション内**で `outbox` テーブルに1行INSERTする。
- outboxテーブルの典型スキーマ:
  - `id`
  - `topic`
  - `message`
  - `state`（`pending`＝未送信 / `processed`＝送信済み、などの状態）
  - `created_at`（挿入時刻。relayがこの順でメッセージを処理する）
  - `processed_at`（relayが送信に成功した時刻）
- 別プロセスの **relay** がoutboxテーブルをポーリングし、`pending` 状態の行をSQSにpublishする。relayはリクエストのライフサイクルとは独立して、独自のリトライロジックを持つ。
- 反対側の **consumer** がSQSからメッセージを受け取り、fulfillmentサービス側のDynamoDBに書き込む。at-least-once配信を前提に、書き込み時に単一の条件チェック（冪等性チェック）を行う。
- 障害耐性: アプリがコミット後・relay実行前にクラッシュしても、outbox行は`pending`のままDBに残っているため、relayは次回イテレーションで確実にそれを拾って送信できる。

### ③ なぜそれが効果的か

「ビジネスレコードの書き込み」と「outboxへの書き込み」を**同一トランザクション**にまとめることで、DBレベルで整合性のギャップを閉じる。ネットワーク呼び出し（SQSへの送信）をリクエストのライフサイクルから完全に切り離し、relay・consumerそれぞれが独自にリトライ／冪等性を扱うことで、分散トランザクション（2相コミット等）なしに「確実に一度は配信される（reliable, decoupled delivery）」仕組みを実現している。

---

## 6. A Deep Dive into How Hono Is Built（Flavio Copes）

- リンク: https://flaviocopes.com/how-hono-is-built/

### ① 目的・問題意識

軽量Webフレームワーク Hono の内部実装を、1つのリクエストが処理される流れに沿って追いかけ、ルーティング・ミドルウェア・Context・Web標準・ランタイムアダプタがどう連携しているかを解説する記事。

### ② 具体的なコード・技術詳細

- **SmartRouterによるルータ選択**: `new Hono()` はデフォルトで `SmartRouter` を使う。最初のリクエスト時に、まず `RegExpRouter` を試し、登録されたパターンに適合しない場合は `TrieRouter` にフォールバックする。一度採用したルータはそのまま保持され、以降のリクエストではルータ選択のオーバーヘッドが発生しない。
- **RegExpRouter**: 互換性のある動的ルートをまとめて最適化された1つのマッチャーへコンパイルする。リクエストごとに登録ルートのリストを1件ずつ照合していくのではなく、あらかじめ結合された正規表現でマッチングする。
- **TrieRouter**: `RegExpRouter` が `UnsupportedPathError` を投げるような複雑なパターンも含め、全パターンを処理できる汎用フォールバック。木構造によりO(k)（kはパスのセグメント数）でのマッチングを提供する。
- 基本的なルート定義の例:
  ```javascript
  import { Hono } from 'hono'
  const app = new Hono()
  app.get('/hello/:name', (c) => c.text(`Hello ${c.req.param('name')}`))
  ```
- **遅延パラメータ抽出（lazy parameter extraction）**、**Contextオブジェクトの生成**、**オニオン（玉ねぎ）方式のミドルウェア合成**（外側から内側へ処理が進み、内側から外側へ戻ってくる、Koaライクな仕組み）を採用。
- `c.text()` のようなレスポンスヘルパーは、標準の `Response` オブジェクトへマッピングされる。
- ランタイムアダプタ（Node.js／Cloudflare Workers／Bun／Deno）は、コアのアプリケーションロジックに触れることなく、境界部分（リクエスト/レスポンスの入出力）だけを各ランタイムの流儀に変換する。

### ③ なぜそれが効果的か

- ルータの選択を初回リクエスト時の1回だけに限定し、以降はキャッシュされたルータを使い回すことで、リクエストごとのオーバーヘッドを最小化している。
- 正規表現ベースの `RegExpRouter` はマッチングを事前コンパイルされた1回の正規表現評価に落とし込めるため高速だが、対応できないパターンもあるため、木構造の `TrieRouter` を汎用フォールバックとして併用するという「速度と汎用性の両立」設計になっている。
- Web標準の `Request`/`Response` を中心に据え、ランタイム差異をアダプタ層に閉じ込めることで、コアロジックを一切変更せずにNode.js・Cloudflare Workers・Bun・Denoなど複数ランタイムへ展開できる。

---

## 7. DeepSeek Harness: DeepSeek's New Node-Powered Agent Harness

- リンク: https://github.com/deepseek-ai/deepseek-harness / https://deepseek.com/harness/en/
- 著者: DeepSeek

### ① 目的・問題意識

Claude Code的な「AIエージェント用ハーネス」を、モノリシックな実装ではなく**完全にプラグインベース**で構築するというアプローチ。モデル・ツール・スキル・セッション・サンドボックス・ストレージ・ループ・スケジューリング・UIまで、あらゆる機能をプラグインとして提供することで、拡張性・組み替え可能性を高めることを狙う。公開初日で3万スター（記事執筆時点ではさらに伸びて19万スター超・2万1400フォーク超）を獲得。

### ② 具体的なコード・技術詳細

- 基盤となっているのは **Cordis** という既存のNode.js向けプラグインシステム。DeepSeekはCordisの作者を採用している。
- Cordisの設計思想は "A Programming Paradigm for Spatiotemporal Composability"（時空間的合成可能性のためのプログラミングパラダイム）に基づく。空間的（複数プラグインの組み合わせ）・時間的（ライフサイクル管理）の両方の次元で、システム全体を再起動せずにプラグインを追加・削除・再構成できる。
- 典型的なCLIハーネスとは異なり、**WebUI経由**で動作する点が特徴。
- npm経由の実行:
  ```bash
  npx @deepseek-ai/dsh web
  ```
  デフォルトで `http://127.0.0.1:3080` にWeb UIが立ち上がり、既定のブラウザで自動的に開く。`--no-open` オプションでブラウザの自動起動を抑制できる。
- ソースからのビルド:
  ```bash
  git clone https://github.com/deepseek-ai/deepseek-harness.git
  cd deepseek-harness
  pnpm install
  pnpm run build
  pnpm dsh web
  ```
- プロジェクトは明示的に "developer preview" と位置づけられており、急速なイテレーションに伴う破壊的変更が予告されている。

### ③ なぜそれが効果的か

「モデル」「ツール」「サンドボックス」「UI」など、通常はハードコードされがちな要素まで全てプラグインとして扱うことで、コミュニティやユーザーが個々のコンポーネントだけを差し替え・拡張できる。これにより、モデル自体の進化が速い領域において「ハーネス（土台）はユーザー・コミュニティが育てていく」という分業が可能になる。

---

## 8. unbash: Fast 0-Dependency Bash Parsing Library

- リンク: https://github.com/webpro-nl/unbash
- 著者: Lars Kappert（Knip・Release It! の作者）

### ① 目的・問題意識

貼り付けられたコマンド・完全なスクリプト・他フォーマットに埋め込まれたシェルソースなど、「Bashのソースコード」を**実行せずに構造を検査したい**というニーズに応える。もともとは未使用コード検出ツール Knip がシェルスクリプトを解析するために使っていたパーサを、単独のライブラリとして切り出したもの。

### ② 具体的なコード・技術詳細

基本的なパース:
```javascript
import { parse } from "unbash";

const ast = parse('if [ -f "$1" ]; then cat "$1"; fi');
```

AST出力の構造例:
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

単語（word）内部のパーツへのアクセス:
```javascript
const word = parse("echo a$(id)b").commands[0].command.suffix[0];
word.parts; // [Literal, CommandExpansion, Literal]
```

ASTをシェルコードへ戻す（プリンタ）:
```javascript
import { print } from "unbash/printer";
const script = print(ast);
// Output: if [ -f "$1" ]; then\n  cat "$1"\nfi
```

技術詳細:
- コマンド、制御フロー、パイプライン、リダイレクト、代入、複合文、パラメータ／単語展開、プロセス置換・入れ子の置換、coproc、heredoc、herestring、ネストされた／生成された構文をサポートし、パラメータオペランドや配列インデックス内部のネストされたコマンドも構造化されたまま保持する。
- 不正な入力・不完全な入力に対しても、ベストエフォートで部分的なASTと、ソース位置付きのエラー情報を返す（パース全体を諦めない）。
- ゼロ依存・同期実行のTypeScript実装。
- パフォーマンス: 短い入力で75.7 MB/s、大きいファイル（965KB）で115.7 MB/sのスループットを実現し、tree-sitter-bashなどの代替手段より11〜17倍高速。

### ③ なぜそれが効果的か

型付き・ソース位置情報付きのASTを返すことで、権限プロンプト用のコマンド分類、CI設定内の実行ファイル／ファイル参照の棚卸し、精密なコマンド書き換え・移行、生成／貼り付けされたBashへの診断付与など、「実行せずに安全にBashを解析したい」多様なユースケースに再利用できる。ゼロ依存・高速であることから、Knipのような他ツールに組み込んでも動作コストが小さい。

---

## 9. njsscan 1.0: A Semantic-Aware SAST Tool for Node

- リンク: https://github.com/ajinabraham/njsscan
- 著者: Ajin Abraham（OpenSecurity / MobSFの作者）

### ① 目的・問題意識

Node.jsアプリケーションにおける安全でないコードパターンを、デプロイ前の静的解析で検出したいというニーズに応えるツール。単純な正規表現ベースのスキャナでは見逃されがちな、文脈依存のロジック上の脆弱性まで検出することを狙う。

### ② 具体的なコード・技術詳細

インストール:
```bash
pip install njsscan
```
要件: Python 3.10以上、Mac/Linux。

基本的な使い方:
```bash
njsscan [path]
njsscan test.js
njsscan . --json -o results.json
njsscan . --sarif --output report.sarif
```

検出対象の脆弱性カテゴリ:
- XSS（信頼できないユーザー入力がHTTPレスポンスに混入するケース）。例: `res.send('<h1> Hello :' + name + "</h1>")` のように `name` がクエリパラメータ由来の未検証入力である場合、反射型XSSとして検出される。
- SQLインジェクション（未検証入力を連結した生SQLクエリ）
- NoSQLインジェクション
- オープンリダイレクト（未検証のリダイレクト先）
- SSRF（サーバーサイドリクエストフォージェリ）
- 安全でない `eval()` の使用
- JWT／トークンの不適切な扱い
- シークレット漏洩（ハードコードされた認証情報・APIキー）
- ExpressアプリにおけるCSRF対策・レート制限・Helmetヘッダなど、セキュリティ設定の欠如

### ③ なぜそれが効果的か

単純パターンマッチ（libsast）と、構文を理解した意味解析（semgrepベースのルール、約70個）を組み合わせることで、単純なアンチパターンだけでなく複雑なロジック上の欠陥も検出できる。GitHub Actions・GitLab・CircleCI・Travisなど主要CI/CDへの統合が可能で、継続的なセキュリティ監視をパイプラインに組み込める。

---

## 10. Hucre 1.0: A Zero-Dependency Spreadsheet Engine

- リンク: https://github.com/productdevbook/hucre / https://hucre.dev/
- 著者: Wind

### ① 目的・問題意識

SheetJS（約300KB）やExcelJS（約500KB）、openpyxlのような既存のスプレッドシート処理ライブラリはバンドルサイズが大きく依存関係も複雑になりがちで、エッジランタイムやブラウザなどリソース制約のある環境には不向き。この課題を解決する、ゼロ依存・軽量なTypeScript製スプレッドシートエンジン。

### ② 具体的なコード・技術詳細

対応フォーマット: 読み書き＝XLSX / ODS / CSV / JSON / NDJSON / XML、読み取り専用＝XLS(BIFF8) / XLSB、書き出し＝HTML / Markdown。

基本的な読み書き:
```typescript
import { readXlsx, writeXlsx } from "hucre"

const workbook = await readXlsx(buffer)
console.log(workbook.sheets[0].rows)

const xlsx = await writeXlsx({
  sheets: [{
    name: "Products",
    columns: [
      { header: "Name", key: "name", width: 25 },
      { header: "Price", key: "price", width: 12, numFmt: "$#,##0.00" },
    ],
    data: [
      { name: "Widget", price: 9.99, stock: 142 },
    ],
  }],
})
```

大容量ファイルのストリーミング処理（一定メモリで処理可能）:
```typescript
for await (const row of streamXlsxRows(buffer)) {
  console.log(row.index, row.values)
}

function* rows() {
  for (let i = 0; i < 5_000_000; i++)
    yield [i + 1, Math.random()]
}

return new Response(
  writeXlsxStream(rows(), {
    name: "BigData",
    columns: [{ header: "ID" }, { header: "Value" }],
  }),
  { headers: { "content-type": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" } }
)
```

データバリデーション（ドロップダウンリスト）:
```typescript
await writeXlsx({
  sheets: [{
    name: "Sheet1",
    rows: [["Status", "Quantity"], ["active", 10]],
    dataValidations: [{
      type: "list",
      values: ["active", "inactive", "draft"],
      range: "A2:A100",
    }],
  }],
})
```

ハイパーリンク:
```typescript
import { writeXlsx, link } from "hucre/xlsx"

await writeXlsx({
  sheets: [{
    name: "Summary",
    columns: [
      { header: "Link", key: "link" },
      { header: "ID", key: "id" },
    ],
    data: [
      { link: link("Open", "https://example.com/abc-123"), id: "abc-123" },
    ],
  }],
})
```

オブジェクト指向のビルダーAPI:
```typescript
const xlsx = await WorkbookBuilder.create()
  .addSheet("Products")
  .columns([
    { header: "Name", key: "name", autoWidth: true },
    { header: "Price", key: "price", numFmt: "$#,##0.00" },
  ])
  .row(["Widget", 9.99])
  .freeze(1)
  .done()
  .build()
```

チャート追加:
```typescript
import { addChart, writeXlsx } from "hucre"

const dashboard = { name: "Dashboard", rows: dashboardRows }
addChart(dashboard, {
  type: "column",
  title: "Quarterly Revenue",
  series: [
    { name: "Revenue", values: "B2:B5", categories: "A2:A5" },
  ],
  anchor: { from: { row: 6, col: 0 }, to: { row: 22, col: 7 } },
})
await writeXlsx({ sheets: [dashboard] })
```

テンプレートへの値埋め込み:
```typescript
const workbook = await openXlsx(templateBuffer)
fillTemplate(workbook, {
  company: "Acme Inc",
  date: new Date(),
  total: 12500,
})
const output = await saveXlsx(workbook)
```

CLI:
```bash
npx hucre convert input.xlsx output.csv
npx hucre convert data.csv report.md
cat data.csv | npx hucre convert - out.xlsx
npx hucre inspect file.xlsx
npx hucre validate data.xlsx --schema schema.json
```

その他の特徴:
- 完全にtree-shakeable（CSVのみのインポートでgzip後約3.7KB、フル機能で129KB）
- ネイティブESM・TypeScript、外部依存ゼロ
- パスワード保護されたワークブック（ECMA-376 Agile暗号化）に対応
- `openXlsx`/`saveXlsx` によるラウンドトリップで、チャート・マクロ・条件付き書式などhucreがネイティブにモデル化していない要素も保持したまま編集可能
- 条件付き書式（13種類のルールタイプ）、ピボットテーブル、スライサー、データバリデーション、ハイパーリンク、画像、アクセシビリティ監査、パスワード暗号化に対応

### ③ なぜそれが効果的か

- Tree-shaking設計により「使う機能分だけ」バンドルに含まれるため、既存の巨大ライブラリに比べて圧倒的に軽量（CSVのみなら3.7KB）。
- 真のストリーミングアーキテクチャにより、利用可能なメモリを超えるサイズのファイルでも一定メモリで処理できる（他のJS系ライブラリにはあまりない特性）。
- `openXlsx`/`saveXlsx`のラウンドトリップ保証により、既存のExcelファイル（チャートやマクロを含む）を安全に編集対象にできる。
- 条件付き書式・ピボットテーブル・暗号化など、通常は商用ライブラリでしか提供されないような機能をカバーしている。

---

## その他の短信（IN BRIEF）

以下は本文中で簡潔に触れられているのみのニュース項目です（コード例は特になし）。

- **Matteo Collina（Node.jsメンテナ）のブログ**: 週に20〜40件寄せられるセキュリティレポート（AIが自動生成したものを含む）をどうトリアージしているかを解説。
- **Bun 1.4 のリリースが間近**: 目玉機能としてNode.js互換性の強化が予告されている（記事執筆時点では未リリース）。
- **`node:domain` モジュール**: ドキュメント上は2015年から非推奨だったが、Node 27で「ランタイムレベルでの非推奨化」が検討されている。利用者は今のうちに意見表明を、とのこと。
- **Node 24.18.1 → 24.19.0 アップグレード後のOOM（メモリ不足）クラッシュ報告**: 複数チームから報告があり、24.18.1へのロールバックで解消するとのこと。原因はまだ確定していない。
- **Vercelのワンクリックアップグレード機能**: 非推奨となったNode.jsバージョンからのアップグレードを支援する機能を提供開始。特にNode 20ベースの新規ビルドは2026年10月1日以降失敗するようになる、との注意喚起。

---

*本まとめは Node Weekly Issue #637（2026年8月13日号）を基に作成しました。次号は夏季休止を経て8月27日発行予定です。*
