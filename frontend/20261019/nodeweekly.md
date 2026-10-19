# Node Weekly Issue #644 (2026-10-15) — The NodeConf EU 2026 talks are online

出典: https://nodeweekly.com/issues/644

> **取得上の制約**: 実行環境では nodeweekly.com および各リンク先へ直接アクセスできず(WebFetch は DNS 解決不可)、WebSearch の要約結果のみを根拠としている。そのため記事本文の完全なコード・デモの逐語的抽出はできていない。確認できた範囲の事実のみを記載し、コードは「公式情報で確認できた部分」と「説明用の例(※)」を区別している。
> 前号 #643 以降の最新号は #644。次号は 10/29(1週休み)。スポンサー記事は除外。

## 1. NodeConf EU 2026 の講演動画が公開
- **①目的・問題意識**: イタリアで開催された NodeConf EU の全講演が YouTube に公開。内容は Node の FFI 機能、Emscripten と WebAssembly、V8 のヒープ管理、パフォーマンスプロファイリングなど。
- **②コード**: なし(動画リンクのため遷移先確認はスキップ)。
- **③効果**: 実装者本人による内部仕様の解説を参照でき、FFI(Node 26.1 で experimental 導入)や V8 ヒープ挙動の理解に役立つ。

## 2. Deno チームが Cloudflare に参加
- **①背景**: 2026-10-09 発表。Ryan Dahl・Bert Belder を含む Deno チームが Cloudflare へ移り、celld のコード/アイデアを workerd にマージして、workerd のセルフホストを一級サポートにする。背景として workerd の Durable Objects は単一インスタンス内でしか動作せずスケールできない課題がある。
- **②詳細**: Deno ランタイムは今後 1 年、月次のバグ修正/セキュリティリリースを継続し、その後 Deno 側のランタイム開発は終了。Deno Deploy は 6 か月後に終了し、有料顧客は Cloudflare Workers への移行支援を受ける。ニュースレターは「その後コミュニティ維持になることを期待」と記載(検索結果では確認できず)。
- **③影響**: Deno 依存プロジェクトは 1 年以内に Node / Workers 等への移行計画が必要。取引形態(買収か否か)は報道間で表現が割れ、公式は金額等を非開示。
- 参考: https://blog.cloudflare.com/deno-joining-cloudflare (検索結果: https://blog.cloudflare.com/deno-joins-cloudflare/)

## 3. Node.js 新リリース
### Node 26.12.0 (Current, 2026-10-14): 設定ファイルが stable
- **①目的**: ランタイム・テスト・watch・permission のオプションを CLI フラグの羅列ではなく設定ファイルで宣言する。
- **②詳細**: フラグが `--experimental-config-file` から `--config-file` に改名され stable 化(旧名はエイリアスとして残る)。パス未指定時はカレントディレクトリの `node.config.json` を探す。`--config-file=path` で任意のファイルを指定可能(PR #66431, Marco Ippolito)。同リリースで子プロセスへ config-file の permission フラグを伝播(PR #66559)、`node:http` に `websocketMask()`/`websocketUnmask()` 追加。
```bash
# ※使い方の例(キー構成の詳細は https://nodejs.org/api/cli.html を参照)
node --config-file=./node.config.json app.js
node app.js   # カレントに node.config.json があれば自動で読込
```
- **③効果**: 設定をリポジトリで共有・バージョン管理でき、環境差による起動オプションのブレを防ぐ。
- https://nodejs.org/en/blog/release/v26.12.0

### Node 24.22.0 (LTS)
- `util.throttle` と `util.debounce` が LTS 系列に追加(イベント頻度制御を依存パッケージなしで実現)。具体的な API シグネチャは検索では確認できず。
### セキュリティ/ライフサイクル
- 10/21 に 26.x / 24.x / 22.x のセキュリティリリース予定(少なくとも 1 件は HIGH)。早めのアップデート準備が必要。
- Node 24 は 10/20 にメンテナンス入り、Node 26 は 10/28 に Active LTS へ。

## 4. AWS Lambda Web Functions(2026-10-12 発表)
- **①目的・問題意識**: 従来 Lambda 上で Express/Fastify を動かすにはイベントハンドラへのアダプタが必要だった。標準的な「ポートで待ち受ける HTTP サーバー」をそのままデプロイできるようにする。
- **②詳細**: Express、Hono、Fastify、Next.js など任意の Node フレームワークが対象。Lambda が設定する環境変数 `AWS_LAMBDA_HTTP_ENDPOINT` のホスト:ポートで listen する。ローカルでは未設定なので `localhost:3000` にフォールバック(AWS ドキュメントのサンプルと同パターン)。デプロイは 1 コマンド(AWS CLI v2.37.9+ との情報あり)。二次情報では、1 実行環境あたり最大 128 同時リクエスト、アクティブ CPU 時間課金、Node.js 24 / Arm64、VPC 対応、5 リージョン。
```js
// ※説明用の例(公式サンプルと同じパターン)
import express from 'express';
const app = express();
app.get('/', (req, res) => res.send('Hello from Lambda Web Functions'));

const [host, port] = (process.env.AWS_LAMBDA_HTTP_ENDPOINT ?? 'localhost:3000').split(':');
app.listen(Number(port), host);
```
```bash
aws lambda-web deploy --name my-web-app --code ./my-app
# 本番では認証を明示する(既定は公開エンドポイントとの報告あり)
#   --auth-type IamAuth
```
- **③効果**: アダプタ不要でコード変更が最小。複数同時リクエストの処理とアクティブ CPU 課金により、I/O 待ちの多い Web アプリのコスト/性能が改善。注意: 既定が公開(認証なし)との二次情報があるため IAM 認証を指定する。
- https://aws.amazon.com/blogs/aws/announcing-aws-lambda-web-functions-for-fast-consistent-web-app-performance-anywhere/

## 5. WebSocket サーバーのベンチマーク(Node / Bun / Deno)
- 記事のリンク先の数値は検索では取得できなかった。一般的な傾向として Bun が優位とする報告があるが、方法論が不明なものが多く、自環境での計測(autocannon 等)が推奨される。具体的数値は未確認のため省略。

## 未取得の項目
本号の他の記事(リンク集の個別項目)は検索結果から内容を取得できず、本ファイルには含めていない。次回実行時または直接アクセス可能な環境での補完が必要。
