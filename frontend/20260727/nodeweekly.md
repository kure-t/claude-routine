# Node Weekly Issue 634 まとめ (July 23, 2026)

> 出典: [Node Weekly Issue #634](https://nodeweekly.com/issues/634) — "Domenic Denicola's modern dev setup"
> ※ nodeweekly.com への直接アクセスは403のため、WebSearch および RSS フィード経由で内容を収集。sponsor表記の記事（ParadeDB, Tiger Data, Zuplo）は除外。

---

## 1. Node.js コアは prototype pollution に対して「堅牢化できない」

**原文リンク:** [No, We Can't Harden Node.js Against Prototype Pollution](https://adventures.nodeland.dev/archive/no-we-cant-harden-nodejs-against-prototype/)（著者: Matteo Collina）

### ① 目的・問題意識

Node.js のセキュリティ/バグ報奨金プログラムには、Node コアに対する prototype pollution の報告が絶えず寄せられる。しかし Node チームの公式スタンスは一貫している。

> "Any prototype pollution in Node.js core is not accepted as a vulnerability in our security/bug bounty program."（Node コアにおける prototype pollution はいかなるものも脆弱性として受理しない）

記事の核心的な主張は、**「prototype pollution の報告には常に同じ前提条件がある」**という点。その前提とは「攻撃者がすでに `Object.prototype` に書き込める状態にある」ことであり、この前提が成立した時点で、個々に見つかった「gadget（悪用チェーン）」はそれ自体が脆弱性なのではなく単なる症状にすぎない。コア・依存ライブラリ・アプリケーションコードのどこかに、悪用可能な gadget は無数に潜んでいる。つまり「この gadget を塞ぐ」対応にモグラ叩き以上の意味はなく、真の防御策は「攻撃者に書き込みを許さない」入口（信頼できない JSON の境界）側にあるべきだという問題意識を提示している。

### ② 具体的な技術詳細・コード

**Primordials（プリミティブ参照の退避）**

Node コア内部は、起動時にビルトインの元の参照（`Array.prototype.push`、`Object.keys` など）を "primordials" として退避させ、ユーザーコードの実行前に確保している。

```javascript
// コア内部のイメージ：ユーザーが Array.prototype.push を書き換えても
// 影響を受けないように、元の関数への参照を直接呼び出す
const { push: ArrayPrototypePush } = Array.prototype;

// arr.push(x) の代わりに:
ArrayPrototypePush(arr, x);
```

これにより、誰かがグローバルをモンキーパッチしても Node コア自体は動作し続ける。しかし primordials が守るのは「Node が壊れないこと」であって「アプリケーションが安全であること」ではない、と記事は明確に区別している。

**真にコアを堅牢化するために必要な作業（＝非現実的であることの説明）**

記事は、コアを本当に汚染済みプロトタイプから守ろうとすると何が必要かを提示する。

- コアが確保するあらゆるオブジェクトを `{ __proto__: null }` にする
- プロトタイプスロットに触れるあらゆる構文（オブジェクトリテラル、spread、`for...in` など）を避ける
- 依存パッケージすべてに同様の対応を強制する

これは Node コアの primordials 対応がすでにぶつかった壁と同じであり、現実的なコストに見合わない。

**推奨される実務的な緩和策（アプリケーション側での対応）**

- **freeze できるものは freeze する**：`Object.freeze(Object.prototype)` 等で書き込み自体を防ぐ
- **信頼できないキーを扱うときは `Object` の代わりに `Map` / `Set` を使う**

```javascript
// 危険: 信頼できない外部キーをそのままオブジェクトのキーにする
const cache = {};
cache[userProvidedKey] = value; // "__proto__" 等が渡されると汚染されうる

// 安全: Map は組み込みプロトタイプチェーンに依存しない
const cache = new Map();
cache.set(userProvidedKey, value);
```

- **信頼できないデータを保持する場合は null プロトタイプのオブジェクトを使う**

```javascript
// __proto__ を含む JSON を parse しても、そのオブジェクト自体に
// Object.prototype を継承させない
const obj = Object.create(null);
Object.assign(obj, JSON.parse(untrustedInput));
```

### ③ なぜそれが効果的か

`Map`/`Set` や `Object.create(null)` は、そもそも「プロトタイプチェーンを経由した継承」という攻撃の前提条件自体を無効化する。primordials のようにコア側で後追いの防御をするのではなく、**攻撃者の書き込みが到達する「信頼境界」の内側でデータ構造そのものの性質を変える**ため、個別の gadget を塞ぐより本質的に有効。記事はこの視点の転換（「gadget を塞ぐ」から「境界で防ぐ」へ）こそが重要な教訓だとしている。

---

## 2. Domenic Denicola の Agentic Coding セットアップ（2026年7月版）

**原文リンク:** [My Agentic Coding Setup, July 2026](https://domenic.me/agentic-coding-setup/)（著者: Domenic Denicola）

### ① 目的・問題意識

複数の AI コーディングエージェントを並行して安全かつ効率的に走らせるための開発環境構築が課題。具体的には次の問題を解決する必要がある。

- エージェントに「承認なし（zero approvals）」の強い権限を与えたいが、ホストマシンを危険に晒したくない（blast radius の制御）
- 複数エージェントが同じプロジェクトで並行作業する際、ファイルやポートが衝突する
- 生成された作業（サーバー、プレビュー）をネットワークをまたいで安全に確認したい
- 環境がリセットされても設定・エージェント定義を失いたくない

### ② 具体的な技術詳細

- **使い捨て（disposable）Linux VM**：Windows ではない環境をエージェントに与え、「zero approvals」ポリシーで運用できるほどの許容範囲（blast radius）に設定。壊れてもいつでも作り直せる前提。
- **Tailscale**：すべてのマシンからネットワークを問わず VM にアクセス可能にし、エージェントが secure-context の開発サーバー（HTTPS必須の機能を使うdevサーバー）を外部に公開できるようにする。
- **Git worktree + Portless**：同一プロジェクト上で複数エージェントが並行作業してもファイル・ポートが衝突しないようにする。各 worktree ごとに専用ポートでサーバーが立ち上がり、衝突が起きない。
- **典型的なフロー**：エージェントが作業を完了 → サーバーを起動 → Playwright でスモークテスト → `https://agents-base.tail234567.ts.net:8443/` のような Tailscale 経由の URL を提示し、そこで成果を確認できる。
- **VS Code の Remote - SSH 拡張機能**：任意の worktree のコードをリモートから直接レビュー可能にする。
- **GitHub リポジトリ**：全プロジェクトの「セーブポイント」として機能し、VM をリセットする必要が生じても復元できる。
- **chezmoi**：`AGENTS.md`、スキル定義、その他設定ファイルを同期・バックアップする dotfiles 管理ツールとして使用。
- **エージェントクライアントの使い分け**：セッションの引き継ぎ・同期のしやすさから ChatGPT デスクトップアプリを併用。Claude デスクトップアプリは同等の使い勝手ではないため、代わりに Claude Code + tmux の組み合わせを利用。

### ③ なぜそれが効果的か

「使い捨てVM + Tailscale」の組み合わせにより、エージェントに強い自律権限（zero approvals）を与えても、被害範囲をVM内に閉じ込めつつ、どこからでも安全にアクセスできるという相反する要件を両立させている。さらに「worktree + Portless」で複数エージェントの並行実行時の資源衝突を構造的に排除し、「GitHubへの退避 + chezmoiによる設定同期」でVMの使い捨て性とスワークフローの継続性を両立させている点が、実務的に優れた設計となっている。

---

## 3. better-sqlite3 13.0 — N-API 移行と新メソッド

**原文リンク:** [better-sqlite3 Releases](https://github.com/WiseLibs/better-sqlite3/releases)（著者: Joshua Wise）

### ① 目的・問題意識

better-sqlite3 はこれまで Node.js の内部 API（NAN / V8 直結）に依存しており、Node.js や Electron のバージョンが変わるたびにネイティブバイナリの再ビルドが必要だった。これを解消し、実行環境をまたいで動作する安定バイナリを提供することが目的。また、実行される SQL を可視化・デバッグする手段が不足していた。

### ② 具体的な技術詳細・コード

- **N-API への移行**：13.0 は初めて N-API 上で動作するバージョン。N-API は ABI 安定版のインターフェースであるため、プリビルドバイナリが Node.js や Electron の異なるバージョン間、さらには Bun のような別ランタイムでも理論上動作するようになる。

```javascript
// db.explain(): バインドパラメータを渡さずに EXPLAIN クエリを実行できる
const db = new Database('foo.db');
const plan = db.explain('SELECT * FROM users WHERE id = ?');
console.log(plan);
```

```javascript
// preparedStatement.toString(): プリペアドステートメントの展開済みSQLを取得
const stmt = db.prepare('SELECT * FROM users WHERE id = ?');
stmt.bind(42); // .bind() で永続的にバインドされたパラメータのみ展開される
console.log(stmt.toString());
// => "SELECT * FROM users WHERE id = 42"
// 注意: .get() などの実行時に渡す一時的なパラメータは展開されない
// （その場合は verbose オプションを使う）
```

### ③ なぜそれが効果的か

N-API 化により「Node.js のマイナーバージョンが上がるたびにネイティブモジュールが壊れる」という better-sqlite3 利用者にとって長年の悩みが構造的に解消され、Bun などの代替ランタイムでの利用可能性も広がる。`db.explain()` と `toString()` は、実行前に実際に発行される SQL を確認できるため、クエリのデバッグやパフォーマンスチューニングを大幅に容易にする。

---

## 4. AsyncAPI npm サプライチェーン侵害の分析

**原文リンク:** [Unpacking the AsyncAPI npm supply chain compromise and import-time payload delivery](https://www.microsoft.com/en-us/security/blog/2026/07/15/unpacking-asyncapi-npm-supply-chain-compromise-import-time-payload-delivery/)（Microsoft Security Research）

### ① 目的・問題意識

2026年7月14日、Microsoft Threat Intelligence が `@asyncapi` npm organization（AsyncAPI 仕様とコード生成に広く使われるパッケージ群）に対する協調的なサプライチェーン侵害を検知。この事件は、正規の CI/CD パイプラインと npm の provenance（来歴）証明という「信頼の仕組み」自体が悪用されうることを示す事例として分析された。

### ② 具体的な技術詳細

- **侵害範囲**：`@asyncapi/specs`（プレリリース版 6.11.2-alpha.1 と正式版 6.11.2 の両方）、`@asyncapi/generator@3.3.1`、`@asyncapi/generator-components@0.7.1`、`@asyncapi/generator-helpers@1.1.1` の計4パッケージ・5バージョンが、約90分の間に連続して不正公開された。これらは合計で週間約290万ダウンロードに達する。
- **侵入経路**：攻撃者は設定不備のある GitHub Actions ワークフローを悪用し、高権限の Personal Access Token（PAT）を窃取。窃取したトークンを使い、**プロジェクト自身の正規リリースワークフロー経由**でパッケージを公開したため、公開されたパッケージは正当な npm OIDC provenance attestation（来歴証明）を保持していた。
- **ペイロードの発火タイミング**：バックドアとなる JavaScript は実行時モジュール内に注入され、**インストール時ではなくインポート時**に実行される。これにより、依存解決が完了した後、ビルド・ドキュメント生成・CI/CDジョブなどの中で「サイレントに」起動する。
- **影響範囲**：`@asyncapi/specs` は多数の AsyncAPI ツール群の推移的依存関係であるため、露出期間中に該当バージョンを解決・インポートした開発者のワークステーション、CI/CDパイプライン、コンテナビルド、本番サービスすべてが影響を受けうる。

### ③ なぜそれが効果的か（攻撃者視点での有効性）

正規のリリースワークフローを乗っ取ったことで、悪意あるコードが**正当な OIDC provenance 証明付き**で配布され、署名や来歴チェックによる検知をすり抜けた。さらに「インストール時」ではなく「インポート時」に発火する設計は、静的なインストールスクリプト監視や `--ignore-scripts` のような従来型の対策を無効化し、実際にモジュールが使われる（import される）タイミングまで検知を遅らせる点で、既存の防御を体系的に迂回している。

---

## 5. Slopsquatting：5つの主要LLMが共通して幻覚するパッケージ名 53件

**原文リンク:** [New Study Identifies 53 Slopsquatting Targets Across 5 Frontier LLMs](https://socket.dev/blog/slopsquatting-targets-across-frontier-llms)（研究: Aleksandr Churilov、"The Range Shrinks, the Threat Remains"）

### ① 目的・問題意識

AI コーディングエージェント／LLMがコード生成時に実在しないパッケージ名を「幻覚（hallucinate）」して提示することがある。攻撃者がその幻覚された名前を先回りして npm/PyPI に登録すれば（＝slopsquatting）、AIの提案をそのまま信じた開発者が悪意あるパッケージをインストールしてしまう。本研究はこのリスクが2026年時点の最新モデル群でどの程度現存するかを定量化した。

### ② 具体的な技術詳細

- 約20万件のコード生成レスポンスを、Claude Sonnet 4.6、Claude Haiku 4.5、GPT-5.4-mini、Gemini 2.5 Pro、DeepSeek V3.2 の5モデルに対して調査。
- 5モデルすべてに共通して出現した実在しないパッケージ名が127件見つかった。
- PyPI Security と Socket によるレビューの結果、2026年4月時点で実際に登録可能（未使用）だったのは53件（PyPI 41件、npm 12件）。
- 研究はこの53件を「actively malicious な攻撃」としてではなく、あくまで「今後 slopsquatting の標的になりうる名前」として報告。悪用の証拠は確認されていない。

### ③ なぜそれが効果的か（脅威としての説明）

複数の独立したLLMが**同じ幻覚パッケージ名**に収束するという事実自体が重要。単一モデルの気まぐれな幻覚であれば標的化は困難だが、5モデル共通で127件、うち53件が実際に登録可能という結果は、「攻撃者はどのAIツールを使う開発者にも一律に効く」パッケージ名を事前に特定できることを意味する。これにより、攻撃者はAIコーディングツール利用者全体を横断的に狙える再現性の高い攻撃面を得る。

---

## 6. BullMQ によるバックグラウンドジョブ処理

**原文リンク:** "Running Background Jobs with BullMQ"（著者: Flavio Copes）※原文の直接内容確認は不可のため、BullMQ公式ドキュメント等から技術内容を補完

### ① 目的・問題意識

メール送信・画像処理・レポート生成など、HTTPリクエストの応答をブロックすべきでない重い処理を、Node.jsアプリケーションの外側（バックグラウンド）で非同期に実行し、リトライやスケジューリングも扱えるようにする。

### ② 具体的な技術詳細・コード

BullMQ は Redis（Redis Streams）を基盤にしたジョブキューライブラリで、TypeScript をネイティブサポートする。

```javascript
// キューへジョブを追加する側（producer）
import { Queue } from 'bullmq';

const emailQueue = new Queue('emails', {
  connection: { host: '127.0.0.1', port: 6379 },
});

await emailQueue.add('welcome-email', {
  to: 'user@example.com',
  templateId: 'welcome',
});
```

```javascript
// ジョブを処理する側（worker）
import { Worker } from 'bullmq';

const worker = new Worker(
  'emails',
  async (job) => {
    const { to, templateId } = job.data;
    await sendEmail(to, templateId);
  },
  { connection: { host: '127.0.0.1', port: 6379 } }
);

worker.on('completed', (job) => {
  console.log(`Job ${job.id} completed`);
});
```

### ③ なぜそれが効果的か

Queue（登録側）と Worker（処理側）をプロセスとして分離できるため、重い処理をWebサーバーのリクエスト/レスポンスサイクルから切り離してスケールさせられる。Redis を永続化層として使うことで、プロセスがクラッシュしてもジョブが失われず、リトライやスケジュール実行、進捗管理を標準機能として扱える点が、自前のキュー実装に比べた優位性となる。

---

## 7. Execa 10.0 — サブプロセス実行APIの強化

**原文リンク:** [execa releases](https://github.com/sindresorhus/execa/releases)（著者: Sindre Sorhus）

### ① 目的・問題意識

Node.js 標準の `child_process` はAPIが低レベルで扱いにくく、ストリーム変換や子孫プロセスの確実な終了処理などを自前で書く必要があった。Execa はこれをヒューマンフレンドリーなAPIとして提供してきたライブラリで、10.0ではストリーム連携とプロセス終了処理を強化。

### ② 具体的な技術詳細・コード

- **`killDescendants` オプション**：シェル経由 (`shell: true`) でサブプロセスがさらに子プロセスを生成するケースで、子孫プロセスも含めて確実に終了させる。

```javascript
import { execa } from 'execa';

const subprocess = execa('some-script.sh', { shell: true, killDescendants: true });
subprocess.kill(); // 子孫プロセスまで含めて終了する
```

- **Web Stream 変換API**：サブプロセスを Web標準の ReadableStream/WritableStream/TransformStream に変換できる。

```javascript
const subprocess = execa('command');
const readable = subprocess.readableStream();
const writable = subprocess.writableStream();
const transform = subprocess.transformStream();
```

- **`subprocess.pipe()` の戻り値強化**：パイプ先（destination）サブプロセスのメソッドを戻り値がそのまま公開するようになり、パイプ結果を直接扱える。

```javascript
const { stdout } = await execa('cat', ['file.txt']).pipe(execa('grep', ['pattern']));
```

### ③ なぜそれが効果的か

Web Stream への変換対応により、Fetch API の `body` や他の Web標準ストリームAPIとサブプロセスの入出力を橋渡しでき、Node固有のStreamに縛られない相互運用性が生まれる。また `killDescendants` はシェル経由で起動した孫プロセスが残り続ける典型的なゾンビプロセス問題を、オプション一つで解消する。

---

## 8. eslint-package-json — package.json 専用のESLintルール集

**原文リンク:** [sindresorhus/eslint-package-json](https://github.com/sindresorhus/eslint-package-json)（著者: Sindre Sorhus）

### ① 目的・問題意識

`package.json` は自由形式のJSONであるため、不正なパッケージ名、壊れた `exports` フィールド、誤ったバージョン範囲指定、`@types/*` の誤配置など、実行時まで気づきにくいミスが多発する。これをLintで機械的に検出・自動修正する。

### ② 具体的な技術詳細

- ESLint >=10.4、flat config、ESM 必須というモダンな前提で構築。
- 検出するルールの例：
  - 無効なパッケージ名
  - 不正な semver バージョン範囲
  - `exports`/`imports` における非推奨の末尾スラッシュ形式のフォルダマッピングの禁止
  - `exports`/`imports` における文字列ターゲットのフォールバック配列の非推奨化
  - `exports` 内の型定義（`types`条件）が正しい順序・モジュール互換性を持つことの強制
  - `@types/*` パッケージの誤った配置の検出
  - `files` フィールドの冗長なエントリの検出
  - 依存関係のアルファベット順ソートの強制
- 多くのルールは自動修正 (`--fix`) に対応。

### ③ なぜそれが効果的か

`package.json` の不備（特に `exports` の条件分岐ミス）は、パッケージを公開してからESM/CJS利用者側で「モジュールが見つからない」といった形で顕在化しやすく、発見が遅れがちな問題。ビルド・公開前の静的な段階でLintとして検出することで、公開後の破壊的な不具合を未然に防げる。

---

## 9. BotKit — スタンドアロンなActivityPubボットを作るフレームワーク

**原文リンク:** [BotKit by Fedify](https://botkit.fedify.dev/)

### ① 目的・問題意識

Mastodon上のボットは、Mastodon自体の制約（1投稿500文字制限など）に縛られる。BotKitは、Mastodonなどの特定プラットフォームのアカウントを必要とせず、Fediverse（ActivityPubネットワーク）上で動作する完全に独立した（standalone）ボット＝それ自体が完結したActivityPubサーバーを、JavaScript/TypeScriptで簡単に構築できるようにする。

### ② 具体的な技術詳細

- 低レベルのActivityPubサーバーライブラリ **Fedify** の上に構築されている。
- API がシンプルなため、1つのTypeScriptファイルだけで完結したボットを書ける。
- メンション、リプライ、フォロー/アンフォロー、引用、投票、絵文字リアクションなど、各種アクティビティに対して非同期関数ハンドラをアサインするだけで応答ロジックを実装できる。
- 各ハンドラは `session` を受け取り、そこから投稿・返信・リアクションなどを行える。
- 現状は **Deno専用**。Node.js / Bun のサポートは今後の予定。

### ③ なぜそれが効果的か

Mastodon API等の特定プラットフォームに依存しないため、投稿文字数などプラットフォーム固有の制約から解放され、Fediverse全体を対象にした独自ルールのボットを作れる。ハンドラベースのシンプルなAPI設計により、フレームワークの学習コストを最小化しつつ、1ファイルで完結する実装が可能な点が開発体験として評価されている。

---

## その他のツールリリース（簡易メモ）

- **Nub 0.5**：`nub init` によるスキャフォールディングを備えたオールインワンツールキット
- **odiff 4.5**：Zig + Nodeバインディングで実装されたSIMD高速画像比較ツール
- **node-html-to-image 6.2**：HTMLからの画像生成ライブラリ
- **find-my-way 9.7**：Fastify採用のradix木ベースHTTPルーター
- **Gitify 7.0**：Git通知をメニューバーに表示するデスクトップアプリ
- **Verdaccio 6.8**：セルフホスト型npmプライベートプロキシ
- **Piscina 5.3**：ワーカースレッドプール実装
- **Prisma 7.9**：ORM最新版

---

## In Brief（短信）

- Node.js セキュリティリリース（7月27日）：26.x / 24.x / 22.x系でHIGH深刻度の脆弱性に対応
- OpenJS Foundation のコントリビューターランキングでMatteo Collinaが首位
- Bunチームがv1.4を予告、「Bun v1.0以来最大のNode.js互換性向上」とアピール
- pnpmがネイティブなワークスペースリリース管理と診断用`doctor`コマンドを追加
- ESLintがv8→v9、v9→v10移行用の公式codemodを提供
