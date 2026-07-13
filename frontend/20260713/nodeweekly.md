# Node Weekly Issue 632 まとめ (July 9, 2026)

> 出典: [Node Weekly Issue 632: npm v12 and TypeScript 7 released](https://nodeweekly.com/issues/632)
>
> 補足: `nodeweekly.com` への直接アクセスは 403 エラーとなるため、本まとめは WebSearch
> による調査（複数の独立した検索結果のクロスチェック）と、記事が指す遷移先ページ
> （npm/TypeScript/Node.js 公式情報・GitHub PR 等）の直接確認により作成した。
> Issue #632 のタイトルストーリーである「npm v12」「TypeScript 7」に加え、同時期
> （2026年7月8日）にリリースされた Node.js 26.5.0 も号内で扱われている強い関連ニュース
> として収録した。スポンサー枠の記事は対象外としている。

---

## 1. npm v12: インストールスクリプトをデフォルトで無効化（サプライチェーン対策）

**原文リンク:** [Preparing for npm v12: install scripts and non-registry sources become opt-in (GitHub Discussions #198547)](https://github.com/orgs/community/discussions/198547) / [npm v12 Ships With Install Scripts Off by Default — Socket.dev](https://socket.dev/blog/npm-12)

### ① 目的・問題意識

近年、npm パッケージの `preinstall` / `install` / `postinstall` スクリプトを悪用したサプライチェーン攻撃（悪意あるコードを `npm install` 実行時に自動実行させる手口）が急増した。これまでは `ignore-scripts=true` を手動設定しない限り、依存パッケージが持つインストールスクリプトが無条件に実行されてしまっていた。npm v12 はこの既定動作を反転させ、「明示的に許可されたスクリプトのみ実行される」という opt-in モデルに変更することで、サプライチェーン攻撃の主要な侵入経路を塞ぐ。

対象は以下の3つの既定値変更：

1. 依存パッケージの `preinstall` / `install` / `postinstall` スクリプトが**デフォルトで無効化**
2. `--allow-git` のデフォルトが `none` に変更（Git 依存が許可なしには解決されない）
3. `--allow-remote` のデフォルトが `none` に変更（HTTPS 経由のリモート tarball 依存が許可なしには解決されない）

`--allow-file` / `--allow-directory` のデフォルトは変更なし。これらの変更はすでに npm 11.16.0 から利用可能で、v12 での挙動を先行して試すことができる。

### ② 具体的なコード・技術詳細

#### 新しい主要コマンド

| コマンド | 用途 |
|---------|------|
| `npm approve-scripts` | 特定の依存パッケージのスクリプト実行を許可し、`package.json` に許可情報を書き込む |
| `npm deny-scripts` | スクリプト実行を明示的に拒否する（`--all` 系フラグより優先して永続化される） |
| `npm approve-scripts --allow-scripts-pending` | 未承認パッケージの一覧を表示する（読み取り専用） |
| `--allow-scripts=<pkg,...>` | グローバルインストールや `npx` 実行時に使用 |
| `--strict-allow-scripts` | npm 11 系で v12 の強制モードを先行して有効化する |

承認は既定でインストール済みバージョンに固定される（例: `pkg@1.2.3`）。`--no-allow-scripts-pin` を付けるとパッケージ名単位で将来のバージョンも許可できる。

#### 既存プロジェクトの移行手順

```bash
npm install
npm approve-scripts --allow-scripts-pending   # 未承認のスクリプトをレビュー
npm approve-scripts --all                     # 現時点で保留中の全スクリプトを承認
git add package.json && git commit
```

これにより、現在の動作を維持したまま、今後追加される「新しい・想定外の」スクリプトからは即座に保護される。

#### ネイティブアドオン（node-gyp）の扱い

`sharp` / `better-sqlite3` / `bcrypt` など、明示的な install スクリプトを持たないパッケージでも `binding.gyp` があると npm が暗黙的に `node-gyp rebuild` を実行する。これも同様にブロック対象となるため、明示的な承認が必要：

```bash
npm approve-scripts        # node-gyp と対象のネイティブパッケージを承認
```

#### Cypress / Playwright / Puppeteer など（postinstall でバイナリDL）

```bash
npm approve-scripts        # cypress / playwright / @playwright/browser-* / puppeteer を承認
```

#### strict モードでの新規パッケージ追加（鶏と卵問題の回避）

```bash
npm install -D esbuild            # インストールはされるが postinstall はスキップ＋警告
npm approve-scripts esbuild       # 承認を記録
npm rebuild esbuild               # 承認済みとして postinstall を実行
```

#### グローバルインストール／npx

```bash
npm install -g --allow-scripts=canvas,sharp <pkg>
# または設定として永続化
npm config set allow-scripts=canvas,sharp --location=user
```

#### `ignore-scripts=true` からの移行

```bash
npm approve-scripts --allow-scripts-pending
```

`.npmrc` に `ignore-scripts=true` が残っている間は一覧表示のみ行われる。許可リストを整備したら `ignore-scripts` を削除する（両方設定されている場合は `ignore-scripts=true` が優先されallowlistを上書きする点に注意）。

#### Soft モード と Strict モードの違い

- **v12 の既定（soft モード）**: 未承認のスクリプトは警告を出しつつスキップされ、インストール自体は継続する
- **`strict-allow-scripts=true`（厳格モード）**: 未承認スクリプトがあるとパッケージ書き込み前にインストールが失敗する

#### パッケージ作者側の対応指針

1. README に `npm approve-scripts` の必要性を明記しリンクする
2. プリビルド済みバイナリを配布する、またはセットアップ処理をユーザーが明示的に呼び出すコマンドに切り出すことで、スクリプト依存自体を減らす
3. 利用者数の多い・影響の大きいパッケージには npm 側から個別に連携の打診が行われる場合がある

### ③ なぜ効果的か

- **既定を deny に反転**することで、依存関係を1つ増やすだけで任意コード実行を許してしまうという、これまでの npm の構造的リスクを根本から塞ぐ
- **バージョン単位のピン留め承認**により、既に承認済みのパッケージが後から悪意あるバージョンに更新された場合でも自動実行されない（サプライチェーン乗っ取り対策）
- **soft モードでの段階導入**が可能なため、既存の大規模モノレポでも「まず全承認 → 後から絞り込み」という無理のない移行ができる
- CI 上で許可リスト（`package.json` への記録）をコミットすることで、**再現可能なビルド**とスクリプト実行の可視化・監査が同時に実現する

---

## 2. TypeScript 7.0 GA: コンパイラを Go に全面移植し 8〜12倍高速化

**原文リンク:** [typescript-go (GitHub)](https://github.com/microsoft/typescript-go) / TypeScript 7.0 GA 関連報道

### ① 目的・問題意識

TypeScript コンパイラ（`tsc`）は14年間、TypeScript で書かれた自身を JavaScript にコンパイルして Node.js 上（V8）で実行する構成だった。この構成には3つの構造的な性能上限があった。

1. **V8 のイベントループはシングルスレッド** — ファイル横断の型チェックを並列化できない
2. **V8 の JIT はウォームアップに時間がかかる** — コンパイラのようなビルドプロセスは、JIT がピーク性能に達する前に処理が終わってしまう
3. **V8 の GC は AST（抽象構文木）のような、長寿命・循環参照を含む木構造データに最適化されていない**

これらの上限を一度に取り払うため、Microsoft はコンパイラ本体を **Go 言語へ全面移植**した（開発コードネーム「Corsa」、ステージング repo が `microsoft/typescript-go`）。TypeScript 7.0 はこのネイティブ Go 実装が正式に GA（General Availability）となったメジャーリリース。

### ② 具体的な技術詳細

#### アーキテクチャ

`typescript-go` リポジトリの構成:
- 言語比率: Go 82.6% / TypeScript 14.2% / JavaScript 3.1%
- `cmd/tsgo/` — CLI 本体
- `internal/` — コンパイラのコア実装（パーサー、型チェッカー、型解決、JSX、宣言emit、ビルドモード）
- `_packages/native-preview/` — プレビュー版の配布パッケージ

Go のネイティブコード実行、共有メモリによるマルチスレッド化、および新しい最適化により、V8 の3つの制約（シングルスレッド／JIT ウォームアップ／GC 特性）を同時に解消している。

#### インストールと使い方

プレビュー期間中（RC 以前）:

```bash
npm install @typescript/native-preview
npx tsgo    # tsc と同じ感覚で使える
```

**TypeScript 7.0 RC 以降は `tsc` コマンド自体がネイティブ実装に置き換わる**ため、既存のビルドスクリプトを変更する必要はない。

```bash
npx tsc --version   # => Version 7.0.x（内部はGoネイティブコンパイラ）
```

#### 機能完成度（プレビュー時点の情報として）

- **完了済み**: パース、型チェック、型解決、JSX、宣言ファイル emit、ビルドモード（`--build` / プロジェクト参照）
- **開発中**: 言語サーバー（LSP、エディタ向け機能）
- **プロトタイプのみ**: ウォッチモード

なお GA 版では TypeScript 6.0 との差分が `CHANGES.md` にまとめられている。

#### パフォーマンス実測値

- ベンチマークとして使われた VS Code 本体（約150万行の TypeScript コードベース）で、フルビルドの `tsc` 実行時間が **77.8秒 → 7.5秒** に短縮（約10倍）
- Microsoft の公表値としては、フルビルドで概ね **8倍〜12倍**の高速化
- 型チェックのみの計測では 125秒 → 11秒台まで短縮という報告もある

#### 既知の制約（移行時の注意点）

- Vue（`vue-tsc`）や Svelte など、TypeScript コンパイラの内部 API に依存するツールチェーンは、ネイティブ Go 実装への対応が追いついておらず、GA 時点ではまだ移行できないケースがある（プラグイン／カスタムトランスフォーマー機構の互換性が課題）
- 上記の「言語サーバー開発中」の通り、エディタの高度な補完・リファクタリング機能は当面 JS 版実装に依存する可能性がある

### ③ なぜ効果的か

- **並列型チェック**により、マルチコア環境で大規模モノレポのビルド時間がリニアに近い形で短縮される
- **JIT ウォームアップが不要**なネイティブコードのため、CI のようにプロセスを毎回起動するワークロードでも高速化の恩恵をフルに受けられる
- **AST に適したメモリ管理**（Go のGCやアリーナ的なメモリレイアウト）により、大規模コードベースでのメモリ使用量・GC 停止時間が改善する
- CLI インターフェース（`tsc`）を維持したまま内部実装だけを差し替えているため、**既存のビルドパイプラインへの影響を最小化**しつつ性能向上を得られる

---

## 3. Node.js 26.5.0 (Current) リリース（2026年7月8日）

**原文リンク:** [Node.js — Node.js 26.5.0 (Current)](https://nodejs.org/en/blog/release/v26.5.0) / [GitHub Release v26.5.0](https://github.com/nodejs/node/releases/tag/v26.5.0)

リリースマネージャー: Richard Lau (@richardlau)

### ① 目的・問題意識

Node.js 26 系（Current ライン）に対する定例マイナーリリース。Web 標準 API（Blob、ESM インポート属性）へのキャッチアップ、パフォーマンス計測機能の強化、TLS 接続の可観測性向上、および複数の暗号関連セキュリティ修正が主目的。

### ② 具体的な技術詳細

#### `Blob.prototype.textStream()`（PR #64036, SEMVER-MINOR）

W3C FileAPI 仕様に準拠し、Blob の内容を UTF-8 デコード済み文字列のストリームとして読み取る新メソッド。内部的には `blob.stream()` を UTF-8 設定の `TextDecoderStream` にパイプするのと等価。

```javascript
const blob = new Blob(['Hello, ', 'Node.js 26.5!'], { type: 'text/plain' });

// 文字列ストリームとして読み取る
const textStream = blob.textStream();
for await (const chunk of textStream) {
  console.log(chunk); // デコード済みの文字列チャンクが得られる
}
```

#### `--experimental-import-text` フラグ（PR #62300, SEMVER-MINOR）

TC39 Stage 3 の「テキストインポート」提案を実装。ESM の import attributes 構文でテキストファイルを文字列としてそのまま読み込める。

```javascript
// --experimental-import-text フラグを付けて実行する必要がある
import text from './file.txt' with { type: 'text' };

console.log(typeof text); // "string"
```

```bash
node --experimental-import-text app.mjs
```

なお、同時に提案されていた「バイトインポート」（`Uint8Array` として読み込む機能）は、V8 側がイミュータブルな `ArrayBuffer` に未対応のため、このPRでは意図的に除外された。Deno は `--unstable-raw-imports` で、Bun はフラグなしでテキストインポートに対応済みで、Node.js は他ランタイムの挙動を見ながら安定化を検討する方針。

#### `perf_hooks`: イベントループ反復ごとのサンプリング遅延（PR #62935, SEMVER-MINOR）

イベントループの各反復（tick）ごとの遅延を計測できるようになり、パフォーマンス計測ツールがより粒度の細かいデータを取得できる。

#### `ReadableStreamTee` の公開（PR #64195, SEMVER-MINOR）

`node:stream` から `ReadableStreamTee`（1つの ReadableStream を2つの独立した読み取り可能なストリームに分岐するユーティリティ）が公開 API として利用可能になった。

```javascript
const { ReadableStreamTee } = require('node:stream');

const [streamA, streamB] = ReadableStreamTee(originalStream);
// streamA と streamB はそれぞれ独立して消費できる
```

#### TLS: ネゴシエートされた TLS グループのレポート（PR #64119, SEMVER-MINOR）

TLS 接続がネゴシエートした鍵交換グループ（耐量子暗号を含む）を直接参照できるようになり、接続の可観測性・デバッグ性が向上。

#### セキュリティ修正

- **crypto**: 巨大な DH ジェネレータのバリデーション追加（PR #64092）
- **crypto**: EdDSA の small-order ポイントを検証時に拒否（PR #64026）
- **zlib**: web streams での gzip の末尾 trailing members を拒否（PR #64023）
- **zlib**: 全ストリーム／brotli ストリームの flush kind バリデーション追加（PR #63746）
- **tls**: X.509 証明書中の巨大な RSA 指数を適切に処理（PR #64093）

#### 依存関係の更新

| パッケージ | バージョン |
|-----------|-----------|
| undici | 8.7.0 |
| nghttp3 | 1.17.0 |
| sqlite | 3.53.3 |

#### その他の注目変更

- `import` によるネイティブアドオンの読み込みがデフォルトで有効化（PR #64221）
- REPL 起動時に `acorn` の読み込みと `vm` コンテキスト生成を遅延させ、起動を高速化（PR #63879）
- `TextEncoder.encode()` で元の文字列コピーを回避しパフォーマンス改善（PR #63897）
- riscv64 で Maglev（V8 の中間 JIT 層）を有効化（PR #62605）
- 新しいリリース署名者として Stewart X Addison (@sxa) が参加（鍵: `655F3B5C1FB3FA8D1A0CA6BDE4A7D232B936D2FD`）

### ③ なぜ効果的か

- **`Blob.textStream()`** により、ファイルアップロード処理などで Blob を都度 `text()` で全読み込みせずに、大きなテキストデータをストリーミングでメモリ効率よく処理できる
- **`--experimental-import-text`** で、設定ファイルやテンプレートなどのテキストアセットを `fs.readFileSync` を介さず ESM の静的解析可能な構文で扱えるようになり、バンドラー連携が容易になる
- **TLS グループの可視化**により、量子耐性鍵交換への移行状況を本番トラフィックで監視・検証できる
- 複数の **crypto/zlib セキュリティ修正**は、不正な入力によるクラッシュや検証バイパスを防ぐため、外部入力を扱うサーバーは早期アップデートが推奨される

---

## まとめ

| トピック | 重要度 | アクション |
|---------|--------|-----------|
| npm v12: インストールスクリプト既定無効化 | ★★★ | `npm approve-scripts --allow-scripts-pending` で早期に許可リストを整備する |
| TypeScript 7.0 GA（Go ネイティブコンパイラ） | ★★★ | `tsc` を7.0系に上げてビルド時間を計測。Vue/Svelte 等プラグイン依存プロジェクトは互換性を先に確認 |
| Node.js 26.5.0 (Current) リリース | ★★☆ | crypto/zlib のセキュリティ修正を含むため、Current ラインを使う環境は早めに追随する |
