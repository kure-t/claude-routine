# Node Weekly Issue 639 まとめ (September 3, 2026)

> **Issue #639: Bun now passes 80.5% of Node's test suite**
> 出典: [https://nodeweekly.com/issues/639](https://nodeweekly.com/issues/639)
> ※ nodeweekly.com への直接アクセスは 403 のため、WebSearch / 各記事の原文（GitHub・公式ブログ等）から内容を再構成した。

---

## 目次

1. [Bun 1.4 — Node.js テストスイート互換率 80.5%、Zig から Rust への全面移行](#1-bun-14--nodejs-テストスイート互換率-805zig-から-rust-への全面移行)
2. [NestJS 12 — ESM ファースト化、Rspack/Vitest 標準化、Standard Schema 対応](#2-nestjs-12--esm-ファースト化rspackvitest-標準化standard-schema-対応)
3. [Checkly — AI エージェントによる Node.js → Go 移植：テストハーネス先行開発](#3-checkly--ai-エージェントによる-nodejs--go-移植テストハーネス先行開発)
4. [srvx 1.0 — Web 標準ベースのクロスランタイム Fetch サーバー](#4-srvx-10--web-標準ベースのクロスランタイム-fetch-サーバー)
5. [SimpleWebAuthn 14.0 — パスキーのポスト量子暗号(PQC)対応](#5-simplewebauthn-140--パスキーのポスト量子暗号pqc対応)
6. [Drydock — npm/PyPI パッケージの公開前サンドボックス差分レビュー](#6-drydock--nmppypi-パッケージの公開前サンドボックス差分レビュー)
7. [Express 生態系のセキュリティ修正（hbs / multer / morgan）](#7-express-生態系のセキュリティ修正hbs--multer--morgan)
8. [その他のリリース・ツール（Zod 4.5, Got 16.0, Restify 12.0, express-rate-limit 8.7 など）](#8-その他のリリースツールzod-45-got-160-restify-120-express-rate-limit-87-など)

---

## 1. Bun 1.4 — Node.js テストスイート互換率 80.5%、Zig から Rust への全面移行

**記事タイトル**: Bun v1.4
**URL**: [https://bun.com/blog/bun-v1.4](https://bun.com/blog/bun-v1.4)（関連: [Bun on X](https://x.com/bunjavascript/status/2079863261630267773)）

### ① 目的・問題意識

Bun は Node.js の「ドロップイン代替」を目指すランタイムだが、これまで実行エンジンは Zig で書かれていた。Zig 自体の言語エコシステムの成熟度・人材確保の難しさ、メモリ確保の効率（アイドル時 CPU 使用率の高さ、メモリの断片化）、そして Node.js との API 互換性のギャップ（テストスイートの通過率が上がりきらない）という 3 つの課題があった。1.4 はこれらを一度に解決するための「最大のジャンプ」と位置づけられている。

### ② 具体的な技術詳細・コード例

**Node.js 互換性**: Node.js の公式テストスイートから **+1,517 件のテストが新たに通過** するようになり、通過率は **80.5%** に到達した（1.3 系からの単一リリースでの最大の伸び）。また 2,900 件以上の issue を修正。

**Zig → Rust 移行の中身**: Bun のコア実行部分が Zig から Rust に書き換えられた。パフォーマンス向上の主因は、JavaScriptCore が独自に使っていた `libpas` アロケータと `mimalloc` を並行運用していた構成を、**mimalloc 単独運用に統合**したこと。Bun 側で mimalloc に以下の拡張を加えている。

- partial page clearing（ページの部分的なクリア）
- スカベンジャースレッド（JS がアイドルのときにバックグラウンドでメモリを解放）
- 遅延ゼロ埋め（lazy zeroing）の改善

**ベンチマーク結果**:
- アイドル時 CPU 使用率: **5倍削減**
- メモリ使用量: **最大 35% 削減**（HTTPサーバー用途では 13%〜48%）
- Linux での起動速度: **50% 高速化**
- 実例: Bun 上で動く Claude Code のプロダクション CPU 使用率が p99 で 24%→10%、p50 で 5.8%→2.5%（約2倍改善）
- ある企業の実運用データ: 9 個の Bun ランタイムサービス全体で平均 CPU が **49.9% 減**、平均ワーキングセットメモリが **60.4% 減**、トレース対象トラフィックは 27.1% 増加

**新規 API**:

```javascript
// Bun.cron() — OS レベルの cron ジョブをスクリプトとして登録
await Bun.cron("./worker.ts", "30 2 * * MON", "weekly-report");

// cron 式のパース（次回実行時刻を取得）
const next = Bun.cron.parse("*/15 * * * *");

// worker.ts 側は Cloudflare Workers の Cron Trigger と同じ形の
// scheduled(controller) ハンドラをエクスポートする
export default {
  async scheduled(controller) {
    // 定期実行したい処理
  },
};
```

```javascript
// Bun.markdown — Markdown → HTML / React / ANSI 変換
const html = Bun.markdown.html("# Hello **world**");     // HTML文字列
const elements = Bun.markdown.react("# Hello **world**"); // React要素
Bun.markdown.render(md, (node) => { /* 要素ごとのコールバック */ });

// ターミナル向けに色付きANSI出力
const ansi = Bun.markdown.ansi(md, { colors: true, hyperlinks: true, wrap: 80 });
```

```javascript
// Bun.WebView — macOS WebKit または Chrome DevTools Protocol(CDP) 経由の
// ヘッドレスブラウザ操作をコア機能として搭載
const view = new Bun.WebView();
await view.load("https://example.com");
const title = await view.evaluate("document.title");
```

```javascript
// Bun.Terminal — node-pty 不要で bash / vim / htop を JS から直接操作できる
// 組み込み疑似ターミナル(PTY)
const term = new Bun.Terminal({ cmd: ["bash"] });
term.write("ls -la\n");
```

```bash
# 並列実行系の新フラグ
bun run --parallel build:a build:b   # 複数スクリプトを並列実行
bun test --parallel                  # テストファイルを並列実行

# サプライチェーン/依存関係管理系の新コマンド
bun audit fix   # 既知の脆弱性がある依存を自動修正
bun dedupe      # 重複した依存関係を統合
bun prune       # 使われていない依存を削除
```

Windows ARM64 対応、そして opt-in のグローバル仮想ストア（依存を1箇所に集約してハードリンクする仕組み）により、インストールが**最大7倍高速化**。

### ③ なぜそれが効果的か

- **アロケータ統合**がボトルネックだったのは、2つの独立したメモリマネージャ（libpas と mimalloc）がそれぞれ別々にOSからページを確保・解放していたため、断片化とアイドル時のポーリングコストが二重に発生していたから。単一アロケータに統合しスカベンジャースレッドで能動的に返却することで、アイドル時のCPU/メモリの双方が同時に改善する。
- Node.js テストスイートを直接の適合基準にすることで、「エッジケースを含めた実動作レベルの互換性」を定量的に追跡できる。+1,517件という具体的な数字は、Node.js からの移行時に「動かないコードがどれだけ減るか」を直接示す指標になっている。
- `Bun.cron` / `Bun.Terminal` / `Bun.WebView` はいずれも、これまで `node-cron` や `node-pty`、Puppeteer のような**外部npmパッケージへの依存を減らす**方向の機能追加であり、依存関係を減らすことでインストール時間・脆弱性表面を削減する狙いがある。

---

## 2. NestJS 12 — ESM ファースト化、Rspack/Vitest 標準化、Standard Schema 対応

**記事タイトル**: NestJS v12 is Now Available
**URL**: [https://github.com/nestjs/nest/releases/tag/v12.0.0](https://github.com/nestjs/nest/releases/tag/v12.0.0)（解説: [Trilon Blog](https://trilon.io/blog/nestjs-12-is-now-available)）

### ① 目的・問題意識

Node.js エコシステム全体が ESM へ移行する中、NestJS はこれまで CommonJS を前提としたアーキテクチャだった。加えてビルドツールとして webpack を使い続けており、モノレポでのビルド速度がボトルネックになっていた。テストランナーは Jest 固定で、Vitest のような高速なランナーへの乗り換えができなかった。バリデーションも `class-validator` に強く依存し、Zod や Valibot など Standard Schema 準拠のライブラリを使いたいユーザーへの対応が薄かった。

### ② 具体的な技術詳細・コード例

**ESM 対応**: コアパッケージはすべて ESM で配布される。Node.js の `require(esm)` 機能により、既存の CommonJS アプリはそのまま動作を継続できる（強制移行ではない）。`nest new` 実行時に CommonJS/ESM のどちらでプロジェクトを生成するか選べるようになった。

**Standard Schema 対応**（Zod / Valibot / ArkType などに対応）:

```typescript
// ルートデコレータに schema オプションを渡せるようになった
@Post()
create(@Body({ schema: CreateUserSchema }) body: CreateUserDto) {
  // CreateUserSchema は Zod などの Standard Schema 準拠オブジェクト
}

// バリデーション・シリアライズ用のパイプ/インターセプター
app.useGlobalPipes(new StandardSchemaValidationPipe());
app.useGlobalInterceptors(new StandardSchemaSerializerInterceptor());

// 既存の class-validator デコレータは引き続きフルサポート
```

**ビルドツール**: `--webpack` / `--webpackPath` フラグ（および `nest-cli.json` の `webpack` / `webpackConfigPath`）は非推奨化され、`--builder rspack` に置き換わる。Rspack は webpack のドロップイン代替として、モノレポのビルド時間を大幅に短縮する。

**テストランナー**: ESM プロジェクトでは Vitest がデフォルトのテストランナーになる（CommonJS プロジェクトは Jest を継続使用）。NestJS 本体・サンプルプロジェクトも Jest から Vitest へ移行済み。デコレータの型サポートは OXC が提供する。

**新規 Observability SDK**:

```typescript
// @nestjs/observe — instrument オプションで手動スパン計装なしに
// HTTP / GraphQL / gRPC / microservices / queue / cron を自動計装
const app = await NestFactory.create(AppModule, {
  instrument: true,
});
```

**CLI 強化**:

```bash
nest upgrade   # v11 → v12 の自動マイグレーション
nest deploy    # Mau 経由のクラウドデプロイ
```

- `oxlint` が新規生成プロジェクトの ESLint を置き換え
- パッケージマネージャとして `bun` を新たにサポート
- ルート衝突を検知する `routeConflictPolicy` オプション
- 例外に機械可読な `errorCode` を付与可能

**破壊的変更**: Node.js 要件は `v20.19+` または `v22.12+`（v21.x系は非対応）。NATS トランスポートは `nats` から `@nats-io/transport-node` に移行。GraphQL サブスクリプションは `subscriptions-transport-ws` が削除され `graphql-ws` が必須に。GraphiQL がデフォルトの GraphQL IDE になり Playground から置き換え。ライフサイクルフックの実行順がコンポーネント階層レベル単位に変更。

### ③ なぜそれが効果的か

- **`require(esm)` を使った段階的移行**により、フレームワーク全体を破壊的にESM化しつつ、既存の巨大なCommonJSコードベースを持つ企業ユーザーが即座に移行を強制されない設計になっている。「新規はESM、既存はCJS継続」という選択制は大規模フレームワークの後方互換性維持パターンとして効果的。
- **Standard Schema** はTypeScriptエコシステムで事実上の標準になりつつあるインターフェースであり、これに乗ることで「class-validatorでなければNestJSで検証できない」というロックインを解消し、Zod/Valibotユーザーを取り込める。
- Rspack はwebpackとほぼ互換のAPIを持ちながらRust実装で高速なため、既存のwebpack設定資産を活かしつつビルド時間だけを改善できる、移行コストの低い高速化手段になっている。

---

## 3. Checkly — AI エージェントによる Node.js → Go 移植：テストハーネス先行開発

**記事タイトル**: Rewriting a Node.js Service in Go With AI Agents
**URL**: [https://www.checklyhq.com/blog/agentic-rewrite-nodejs-to-go/](https://www.checklyhq.com/blog/agentic-rewrite-nodejs-to-go/)

### ① 目的・問題意識

Checkly の "Results Daemon"（1日あたり約9,200万メッセージを処理するコンポーネント）は Node.js で書かれていたが、実行ポッド数やDB負荷の観点でスケールの限界に近づいていた。Go に移植すればリソース効率は改善するはずだが、**AIエージェント（Claude Code）に移植作業自体を行わせる**場合、「人間のレビュアーが最終防波堤」という従来のワークフローが機能しない。エージェントが生成したコードが正しいかどうかを、人間が1行ずつ読むのではなく**機械的に検証できる仕組み**が必要という問題意識。

### ② 具体的な技術詳細

**テストハーネスをコードより先に作る**: エージェントにコードを書かせる前に、まず「何が正しい」かを定義するテストハーネスを構築した。これは移植元（Node.js版）を実際に稼働させて生成した入出力の記録＝**ゴールデンファイル**を正解データとして使う手法。

**ブラックボックス方式**: ハーネスは検証対象システムを完全なブラックボックスとして扱う。ハーネス自身のコードや言語と、検証対象（Node.js実装であれ、後のGo実装であれ）との間に一切の結合を作らない。つまり「入力を渡して出力を比較する」以外の手段でテストを書かない。

**非決定的なフィールドの扱い**: UUID やテスト実行時に生成されるタイムスタンプなど、実行毎に値が変わるフィールドはゴールデンファイル内で `<uuid>` や `<timestamp>` のようなプレースホルダとして記録し、値そのものは比較せず**型だけはチェックする**ことで、本来無視すべき差分が紛れ込むリスクを最小化する。

**境界(boundary)の分離**: データベース・キュー・キャッシュ・他サービスなどの周辺コンポーネントは「境界」として明確に分類し、ハーネス側が厳密に管理する。検証対象システムはこれらに対して**環境変数経由でのみ**接続先を指定される（つまりテスト時は境界をハーネスが用意したスタブ/実体に向ける）。

**成果**: Go版はゼロインシデントでリリースされ、稼働ポッド数が **70%削減**、DB負荷も軽減。エージェント駆動開発が機能した鍵として、「Goの強い型システムがJavaScriptよりもエージェントに適していた」ことが挙げられている（型があることでエージェントの誤りが早期にコンパイルエラーとして検出され、リグレッションへの保護になる）。

### ③ なぜそれが効果的か

- ゴールデンファイル方式は、**「正しさ」の定義を人間の言語による仕様書ではなく、実際に動いているシステムの入出力という客観的事実**に落とし込む。これによりエージェントが生成したコードの正誤判定を自動化でき、人間のレビュー負荷を「テスト結果を見る」だけに削減できる。
- ブラックボックス化・境界の分離は、テストハーネス自体がリライトの対象言語やアーキテクチャに依存しないようにする設計であり、Node.js→Go のような言語横断の書き換えでも同じハーネスを再利用できる。
- 非決定的値のプレースホルダ化は、テストのフレーキーさ（実行毎に失敗/成功が変わる不安定さ）を除去しつつ、「型は合っているが値は無視してよい」という粒度の検証を可能にする、実務上重要なテクニック。

---

## 4. srvx 1.0 — Web 標準ベースのクロスランタイム Fetch サーバー

**記事タイトル**: srvx — Universal Server
**URL**: [https://github.com/h3js/srvx](https://github.com/h3js/srvx)（[README](https://github.com/h3js/srvx/blob/main/README.md)）

### ① 目的・問題意識

Node.js・Deno・Bun のそれぞれで、HTTPサーバーの起動方法・リクエスト/レスポンスの型が異なる（Node.js は `http.IncomingMessage`/`ServerResponse`、Deno/Bun は Web標準の `fetch(Request): Response`）。この差異のせいで、ランタイムを横断して動くライブラリやツールを作るのが難しい。srvx はこのギャップを埋め、**単一のfetchハンドラAPIで3つのランタイム全てに対応するサーバー**を提供する。

### ② 具体的な技術詳細・コード例

```javascript
// 最小のサーバー定義 — Web標準のRequest/Responseのみを使う
export default {
  fetch(req) {
    return Response.json({ hello: "world!" });
  },
};
```

```bash
# 同じコードを3つのランタイムそれぞれで起動できる
npx srvx            # Node.js
deno -A npm:srvx     # Deno
bunx --bun srvx      # Bun

# CLIから直接ハンドラを叩くこともできる
npx srvx fetch /api/users
```

**Node.js 互換レイヤ**: srvx サーバーは内部で `fetch(Request) => Response` 形式のハンドラを `node:IncomingMessage => node:ServerResponse` 形式に変換して、Node.jsの組み込みHTTPサーバーと接続する。逆に、既存のExpressなどNode.js形式のハンドラをWeb標準のfetchハンドラに変換したい場合は、`srvx/node` が提供する `fetchNodeHandler` や `toFetchHandler` を使う。

**特徴**: ゼロ依存、Node.js上でも「ネイティブに近い性能」、Deno/Bunでは実質オーバーヘッドゼロ。ウォッチャー、エラーハンドラ、静的ファイル配信、ロガーを備えたフル機能CLIも同梱。

### ③ なぜそれが効果的か

- Web標準の `Request`/`Response` を共通言語にすることで、ランタイムごとの差異を**アダプタ層に閉じ込め**、アプリケーションコードはランタイム非依存に書ける。これはNitro/H3などのエコシステム全体で採用が進んでいる設計方針と一致する。
- `fetchNodeHandler` のような相互変換ユーティリティを用意することで、Express等の既存Node.js資産を全て書き換えずに、新しいWeb標準ベースのミドルウェアと共存させられる（段階的移行が可能）。

---

## 5. SimpleWebAuthn 14.0 — パスキーのポスト量子暗号(PQC)対応

**記事タイトル**: Release v14.0.0 - The one after they go quantum
**URL**: [https://github.com/MasterKale/SimpleWebAuthn/releases/tag/v14.0.0](https://github.com/MasterKale/SimpleWebAuthn/releases/tag/v14.0.0)

### ① 目的・問題意識

WebAuthn/パスキーはこれまで ECDSA(ES256) や RSA といった古典計算量に依存した署名アルゴリズムを主に使っていたが、量子コンピュータが実用化された場合これらは破られる可能性がある。IANA が COSE仕様にポスト量子アルゴリズムを追加したことを受け、認証器・クレデンシャルマネージャ側の対応が進みつつある中、サーバー側ライブラリとしてこれをいつ・どう扱うかが課題だった。

### ② 具体的な技術詳細

**PQC対応**: `@simplewebauthn/server` が **ML-DSA-44 / ML-DSA-65 / ML-DSA-87** という3つのPQC(Post-Quantum Cryptography)アルゴリズムに対応（対応ランタイム限定）。対応環境では `generateRegistrationOptions()` が呼ばれた際、将来のRelying Party（サービス提供者）をPQC対応認証器の登場に備えさせるため、**自動的に ML-DSA-44 でのパスキー登録を優先的に促す**。

**その他の変更**:
- `@simplewebauthn/browser` に `sendSignal()` メソッド追加 — WebAuthnのSignal API群（クレデンシャルの状態をブラウザ/OSに通知する仕組み）を単一メソッドから呼び出せるように統一
- `browserSupportsPasskeys()` がより高度なWebAuthn検出を実施
- `getBrowserCapabilities()` がWebAuthnの機能検出をラップして追加チェックを行う
- `verifyAuthenticationResponse()` に `expectedTopOrigin` オプション追加（クロスオリジン認証対応）
- `MetadataService.initialize()` にカスタムログ出力用の `logger` オプション追加
- `transports` の型が `string[]` に統一され呼び出しが簡素化
- TPM Attestation検証でメーカーIDの大文字小文字ゆれに対応
- 登録・検証関数間で公開鍵アルゴリズムのデフォルトリストを統一
- `validateCertificatePath()` がx5c配列内のクロス署名証明書に対応

**破壊的変更**: 最小サポートバージョンが **Node.js LTS 22.x以上**、**Deno v2.4.x以上**に上昇。

### ③ なぜそれが効果的か

- 現時点でPQC対応認証器はまだ市場にほとんど存在しないが、**サーバー側ライブラリを先行して対応させておく**ことで、将来対応デバイスが登場した瞬間からRelying Party側の追加対応なしにPQCパスキーを受け入れられる。認証エコシステムはクライアント・サーバー両方の対応が揃わないと機能しないため、「後方互換を保ったまま先行して土台を作る」戦略は現実的。
- `ML-DSA-44` を自動的に優先させる設計は、開発者が明示的にPQCを意識しなくても徐々に量子耐性のあるクレデンシャルへ移行が進む「seamless upgrade path」を作っている。

---

## 6. Drydock — npm/PyPI パッケージの公開前サンドボックス差分レビュー

**記事タイトル**: Two Places to Stop a Bad Release / Drydock Package Review
**URL**: [https://jovidecroock.com/blog/drydock-release-defenses/](https://jovidecroock.com/blog/drydock-release-defenses/) / [https://drydock.org/](https://drydock.org/)

### ① 目的・問題意識

npm/PyPIのサプライチェーン攻撃（悪意あるコードを含むパッケージの公開）が近年頻発しており、2026年8月には ChainDrop と呼ばれる自己増殖ワームが400以上のnpmパッケージに感染する事件も発生している。攻撃者は典型的に「`postinstall`スクリプトでの資格情報窃取」「ネットワーク送信」「新規バイナリの追加」といった手口を使うが、これらは**パッケージが実際にnpm/PyPIに公開されるまで**検知されないことが多かった。Drydockは公開の「手前」でこれを止めることを目的とする。

### ② 具体的な技術詳細

**2つの導入モード**:

1. **Stage Watchtower（npm向け）**: npmの「ステージング→2FA承認」という公開フローに割り込む形で動作。ユーザーが手動でスキャンをトリガーするか、Drydockのcronジョブがステージされた新バージョンを検知すると、そのtarballを取得し、サンドボックス内で展開、直前に公開されているバージョンと比較し、差分レポートをメンテナのダッシュボードに表示。npmの最終承認（2FA）が完了する前にレビューできる。
2. **Workflow Gate（PyPIやCIから直接公開する場合向け）**: CIがリリースアーティファクトを先にビルドし、GitHub Actionsのアーティファクトとしてアップロード。公開ジョブはDrydockが保護するGitHub Environmentに入り、そこでゲートされる。

**検知するシグナル**: 新規の`preinstall`/`postinstall`ライフサイクルスクリプトの追加、以前存在しなかったファイルの追加、プロセスを起動するコードやネットワークにアクセスするコードの追加、新規バイナリの追加など。「マルウェアが依存する典型的な挙動」に的を絞ってフラグを立てる。

**セキュリティ設計上の重要な特性**: サンドボックス内ではパッケージのバイト列を**「敵対的な証拠」として扱う**。サンドボックスはnpmトークンを一切受け取らず、パッケージのコードは実行されない（静的な差分比較のみ）。認証情報は、レジストリへのエンドポイントに必要な狭いゲートウェイ経由でのみ付与される。

### ③ なぜそれが効果的か

- 従来の「公開後にセキュリティスキャナが検知する」方式は、その間に一度でもインストールされれば被害が発生してしまう。Drydockは**公開前**の最終ゲートに位置するため、悪意あるバージョンがエンドユーザーに届く前に止められる。
- パッケージコードを実行せず静的diffのみで判定する設計は、「マルウェアを検知しようとして検知器自身がマルウェアを実行してしまう」というセキュリティスキャナ特有のジレンマを回避している。
- 「差分に紐づくフラグ」という提示方法（変更された行にリスク信号をピン留めする）は、メンテナが「何が新しく追加されたのか」を1行単位で即座に把握できるため、レビューの認知負荷を大きく下げる。

---

## 7. Express 生態系のセキュリティ修正（hbs / multer / morgan）

**記事タイトル**: August 2026 Security Releases
**URL**: [https://expressjs.com/en/blog/2026-08-31-security-releases/](https://expressjs.com/en/blog/2026-08-31-security-releases/)

### ① 目的・問題意識

Expressチームが管理する周辺パッケージ（テンプレートエンジンの `hbs`、アップロード処理の `multer`、アクセスログの `morgan`）で、それぞれ独立した脆弱性が発見された。テンプレート出力のエスケープ漏れ、アップロード処理でのクラッシュ、ログ改ざんという3系統の問題を1回のセキュリティリリースでまとめて対応。

### ② 具体的な技術詳細

- **hbs 4.3.0（XSS）**: `hbs` 2.1.0〜4.2.1 において、`registerAsyncHelper` で登録された非同期ヘルパーは、同期ヘルパーと異なり出力がエスケープされない実装になっていた。ユーザー入力に依存する値を非同期ヘルパー経由で描画しているアプリでは、任意スクリプト実行(XSS)につながる。
- **multer 2.3.0（DoS）**: マルチパートリクエストで特別に作り込んだテキストフィールド名を送ると、multerのフィールドパーサ内で未捕捉の `RangeError` が発生し、Node.jsプロセスが終了してしまう（＝リモートから引き起こせるDoS）。2.3.0ではこのパースエラーを捕捉し、クラッシュではなくリクエストの拒否に変更。加えて、`limits.fieldArrayIndexLimit` という opt-in オプションを新設し、配列インデックスが異常に大きいフィールドを拒否できるようにした（アプリが実際に必要とする最大インデックス数を設定することが推奨されている）。
- **morgan 1.12.0（ログ偽装）**: 1.12.0より前のバージョンは、Unicode の行区切り文字（U+0085, U+2028, U+2029）をエスケープせずログに書き込んでいたため、ログの見た目上の改行を偽装され、ログ解析やSIEMでの検知を回避される「ログインジェクション」が可能だった。

### ③ なぜそれが効果的か

- multerの修正は「エラーを握って握りつぶす」のではなく「パースエラーとして正しく扱い、リクエストを拒否する」という設計になっている点が重要。クラッシュ(プロセス終了)をリクエスト単位の失敗に格下げすることで、1リクエストの異常入力がサービス全体を落とすことを防ぐ。
- `fieldArrayIndexLimit` のような **opt-in の上限設定**は、後方互換性を壊さずに「デフォルトでは無制限だが、アプリ側が明示的に絞れる」余地を残す典型的な安全策のパターン。
- morganの修正は、ログというセキュリティ監査で信頼される情報源自体が攻撃対象になり得ることを示す例で、「ログに書き込む文字列は常にエスケープする」という原則を実運用のツールに落とし込んだもの。

---

## 8. その他のリリース・ツール（Zod 4.5, Got 16.0, Restify 12.0, express-rate-limit 8.7 など）

### Zod 4.5 — `z.compile()` によるスキーマの事前コンパイル

**目的**: 大きなオブジェクト・配列・Union型のスキーマは、実行時に型チェックのための分岐が多く、パース速度が問題になることがあった。

**コード**:
```javascript
import * as z from "zod";

const Player = z.object({
  username: z.string(),
  bio: z.string(),
  xp: z.number(),
  // ...20 more properties...
});

// スキーマ専用のバリデーション関数を事前生成する
const CompiledPlayer = z.compile(Player);

Player.parse(data);          // 通常のパース
CompiledPlayer.parse(data);  // 同じAPIで ~9倍高速

// strict指定で、コンパイル不可能な場合に例外を投げさせることもできる
const CompiledOrderSchema = z.compile(OrderSchema, { strict: true });
// → 非対応の場合 ZodCompileAsyncError / ZodCompileUnsupportedError
```
他にも `z.creditCard()`（Luhnチェック付きのクレジットカード番号検証）、`z.validate(): boolean`（パース結果を捨てて真偽値だけ返す高速パス、無効な入力に対して最大16倍高速）、スキーマ1個あたりのメモリを7.5KB→784バイトに削減（メソッドの遅延バインディングによる）などの改善が入った。

**効果**: `z.compile()` は「同じスキーマを繰り返し使う」ホットパス（APIサーバーのリクエストバリデーションなど）で特に有効。オプトインのAPIなので既存コードに影響を与えない。

### Got 16.0 — HTTP/2クライアントの内製化

`http2-wrapper` への依存を廃止し、HTTP/2クライアントをGot内部に書き直した。ALPNネゴシエーション、セッションプールによる多重化、GOAWAY処理、リクエスト/レスポンストレーラー、1xx informationalレスポンス、AbortSignal対応、レスポンスキャッシュ、IPv6アドレス対応、`h2session`経由のh2cをサポート。破壊的変更として、`agent.http2` はエージェントスロットではなく単純なopt-outフラグ（`false`でセッションプーリングを無効化）になり、レスポンスヘッダーにHTTP/2の疑似ヘッダー（`:status`など）が含まれなくなったため `response.statusCode` を使う必要がある。バグの多かったHTTP/2プロキシサポートは削除された。

### Restify 12.0 / np 12.1 / express-rate-limit 8.7

- **Restify 12.0**: node-restify のメジャーバージョンアップ。詳細な変更点は公式CHANGELOGに記載されているが、今回の検索では個々の変更内容までは確認できなかった。
- **np 12.1**: npmパッケージの公開作業を対話的に自動化するCLIツールの最新版。
- **express-rate-limit 8.7.0**: Express用レートリミットミドルウェアの最新パッチ。

### htmx 4.0 / 2026 Node.js ユーザー調査

htmx 4.0のリリース、ライブラリのバージョニングに関する議論、そして2026年のNode.jsユーザー調査（Node財団によるエコシステム利用実態調査）についても本issueで触れられているが、これらは既存機能の延長・アンケート企画であり、新規の「技術」としての深掘りは対象外とした。

---

## 除外した項目

- **TimescaleDB のスポンサー記事**（Postgres拡張によるスローな分析クエリ処理の解説）は、記事末尾に "sponsor" 表記があったため対象から除外した。

---

## 出典一覧

- [Node Weekly Issue #639](https://nodeweekly.com/issues/639)
- [Bun v1.4 Blog](https://bun.com/blog/bun-v1.4)
- [NestJS v12.0.0 Release](https://github.com/nestjs/nest/releases/tag/v12.0.0)
- [Checkly: Rewriting a Node.js Service in Go With AI Agents](https://www.checklyhq.com/blog/agentic-rewrite-nodejs-to-go/)
- [srvx README (GitHub)](https://github.com/h3js/srvx)
- [SimpleWebAuthn v14.0.0 Release Notes](https://github.com/MasterKale/SimpleWebAuthn/releases/tag/v14.0.0)
- [Drydock: Two Places to Stop a Bad Release](https://jovidecroock.com/blog/drydock-release-defenses/)
- [Drydock Package Review](https://drydock.org/)
- [Express.js August 2026 Security Releases](https://expressjs.com/en/blog/2026-08-31-security-releases/)
- [Zod 4.5 Blog: Introducing z.compile()](https://zod.dev/blog/introducing-z-compile)
- [Got Releases (GitHub)](https://github.com/sindresorhus/got/releases)
