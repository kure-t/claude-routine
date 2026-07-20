# Node Weekly Issue 633 まとめ (July 16, 2026)

> 出典: [Node Weekly Issue 633: The life of a ~200ms HTTP request to a Node app](https://nodeweekly.com/issues/633)
> RSS: https://cprss.s3.amazonaws.com/nodeweekly.com.xml
>
> ※ nodeweekly.com への直接アクセスは403となるため、WebSearch/RSSフィード経由で記事を特定し、各リンク先の一次情報を調査した。
> ※ "sponsor" 表記の記事（Master.dev「Harness Engineering」、Tiger Data/TimescaleDBの回）は対象外。

---

## 1. インタラクティブ可視化: 1本のHTTPリクエストの~200msの一生

**原文リンク:** [200ms.thenodebook.com](https://200ms.thenodebook.com/)（The Node Book チーム制作）
**執筆:** Ishtmeet Singh

### ① 目的・問題意識

Webページが表示されるまでの約200msの間に、DNS解決・TCP・TLS・カーネル・Node.jsのイベントループ・Postgresクエリという膨大な機構が連鎖的に動いている。ブラウザの開発者ツールのWaterfallタブを見ても「DNS: 20ms」のような数値しか分からず、その裏で実際に何が起きているかは開発者に見えない。本コンテンツは、コーヒーショップでユーザーがボタンをクリックしてからレスポンスが画面に描画されるまでの全経路を、実際の技術的挙動に基づいて可視化し、言語化する教育コンテンツ。

### ② 具体的な技術詳細（可視化されるフェーズ）

「スクロール量 = 経過時間」という設計になっており、スクロールを止めるとリクエストがその瞬間で「フリーズ」する演出。各レイヤーが1つの「幕（act）」として描かれ、以下の順で進行する。

1. **DNS解決** — ドメイン名がIPアドレスに変換される過程
2. **TCPハンドシェイク** — 実際のSYN → SYN-ACK → ACKのバイト列レベルでの説明
3. **TLSネゴシエーション** — 鍵交換・証明書検証のプロセス
4. **Linuxカーネル** — パケット受信時にカーネルがネットワークソフト割り込み（softirq）をスケジューリングする様子
5. **Node.jsのイベントループ** — libuvが非同期コールバックを拾い上げ、V8に処理を引き渡す流れ
6. **Postgresへのクエリ** — Postgresのパーサーがクエリを解析し実行計画を立てるところまで
7. **応答がブラウザに戻り、画面に描画されるまで**

各フェーズには、実際のプロトコル/カーネル/ランタイムの内部動作（SYN-ACKの正確なバイト列、softirqのスケジューリング、libuvのコールバックキュー処理など）が注釈として付与されている。

### ③ なぜ効果的か

- 「スクロール=時間経過」というインタラクションにより、普段は一瞬で終わる200msという時間感覚を「体感」できる
- DNS/TCP/TLS/カーネル/Node/DBという普段は別々に学ぶレイヤーを、1つの連続したストーリーとして統合的に理解できる
- 抽象的な「レイテンシ内訳」の数字ではなく、各レイヤーで実際に何のコードやプロトコル処理が動いているかを可視化することで、パフォーマンスチューニング時にボトルネックがどの層にあるかを直感的に判断しやすくなる

---

## 2. Fetch APIにはエラーコードが必要だ

**原文リンク:** [Fetch Needs Error Codes — James M Snell](https://www.jasnell.me/posts/fetch-needs-error-codes)

### ① 目的・問題意識

現在の `fetch()` API は、性質の異なる失敗（DNS解決失敗、TLSエラー、サーバーがリクエストを拒否、ストリームがキャンセルされた等）を、区別なくすべて同じ `TypeError` に潰してしまう。これにより、本来は安全にリトライできるはずの情報（＝リトライ可能性の保証）が失われている。例えば HTTP/2 の `REFUSED_STREAM`（サーバーがPOSTリクエストを一切見ていないことを示す信号で、本来リトライして良い）と、DNS解決失敗が、アプリケーションコードから見ると全く同じ `TypeError` になってしまう。

### ② 具体的な技術詳細

提案されているエラーコードは、大きく4つのカテゴリに分類される。

- **rejected（拒否）**: サーバーがリクエストを一切処理していないことが確定しているケース
- **cancelled（キャンセル）**: ストリームが不要になったが、サーバー側で部分的に処理が進んでいた可能性があるケース
- **no-error**: 正常系のクローズ
- **internal-error**: 内部エラー

具体的なコード案:

| エラーコード | 意味 |
|---|---|
| `ERR_HTTP_REQUEST_REJECTED` | サーバーがリクエストを一切見ていない（HTTP/2の `REFUSED_STREAM` 相当）。**最も価値が高い**コードで、非冪等な `POST` リクエストの安全な自動リトライを可能にする |
| `ERR_HTTP_REQUEST_CANCELLED` | ストリームは不要になったが、サーバーが部分的に処理していた可能性がある |
| `ERR_HTTP_GOAWAY` | `ERR_HTTP_REQUEST_REJECTED` と同じリトライ可能性を持つが、ストリーム単位ではなくコネクション単位（`last-stream-id` より上のリクエストに対して）で発生する |
| `ERR_HTTP_DNS_RESOLUTION` / `ERR_HTTP_TLS_ERROR` | 接続確立前のエラー。インフラの内部情報（DNSサーバー、TLS設定など）を露出しうるため、通常のWebコンテキストでは詳細をリダクト（伏せ字化）すべきとされる |

ストリームレベルのコード（`ERR_HTTP_REQUEST_REJECTED` など）は、接続確立後にサーバー自身が明示的に伝えてきた情報を反映しているため安全に公開できるが、接続確立前のコードはインフラ詳細を漏らすリスクがあるという非対称性が論点になっている。

### ③ なぜ効果的か

- `ERR_HTTP_REQUEST_REJECTED` が識別できれば、これまで危険とされてきた **非冪等（POSTなど）リクエストの自動リトライ** を安全に実装できるようになる
- エラーの意味（リトライ可能か、キャンセルされただけか、サーバー拒否か）をアプリケーションコードが判別できるようになり、リトライロジックやサーキットブレーカーの実装が正確になる
- 接続前エラーと接続後エラーを区別してリダクションポリシーを変えることで、セキュリティ（内部インフラ情報の非開示）と開発者体験（エラーの意味の明確化）を両立できる

---

## 3. Proxy と Reflect: オブジェクトの根本的な動作を書き換える

**原文リンク:** [Proxy and Reflect — Piccalilli](https://piccalil.li/blog/proxy-and-reflect/)
**執筆:** Mat Marquis

### ① 目的・問題意識

JavaScriptの `Proxy` と `Reflect` は、ES6（ES2015）からある機能でありながら見過ごされがちな存在。オブジェクトに対するプロパティ取得・設定・削除・関数呼び出しといった「内部操作（internal methods）」そのものを横取り（インターセプト）し、カスタムの振る舞いを差し込めるようにする、JavaScriptの内部動作を覗き込むための強力な仕組みである。Vueなどのフレームワークのリアクティブシステムも内部でこの機構を利用している。

### ② 具体的なコードと技術詳細

#### 基本の `get` / `set` トラップ

```javascript
const target = {
  message: "hello",
};

const handler = {
  get(target, property, receiver) {
    console.log(`プロパティ "${property}" が読み取られました`);
    return Reflect.get(target, property, receiver);
  },
  set(target, property, value, receiver) {
    console.log(`プロパティ "${property}" に ${value} が設定されました`);
    return Reflect.set(target, property, value, receiver);
  },
};

const proxy = new Proxy(target, handler);

proxy.message;          // "プロパティ "message" が読み取られました" → "hello"
proxy.message = "world"; // "プロパティ "message" に world が設定されました"
```

#### なぜ `Reflect` が必要か

トラップの中で `target[property] = value` のように素朴に書くこともできるが、それでは `receiver`（実際にアクセスされたオブジェクト、継承チェーンがある場合は `target` と異なる）を正しく扱えない。`Reflect` の各メソッドは対応する `Proxy` トラップと全く同じシグネチャを持ち、「本来の（デフォルトの）動作」をそのまま呼び出せるガードレールとして機能する。これにより、トラップ内でカスタムロジック以外の部分は常にJavaScriptの仕様通りの挙動を保証できる。

#### その他の主要なトラップ

```javascript
const handler = {
  has(target, property) {
    console.log(`"in" 演算子で "${property}" がチェックされました`);
    return Reflect.has(target, property);
  },
  deleteProperty(target, property) {
    console.log(`プロパティ "${property}" が削除されます`);
    return Reflect.deleteProperty(target, property);
  },
  apply(target, thisArg, args) {
    console.log(`関数が呼び出されました。引数: ${args}`);
    return Reflect.apply(target, thisArg, args);
  },
  construct(target, args) {
    console.log(`"new" でインスタンス化されました`);
    return Reflect.construct(target, args);
  },
};
```

- `has`: `in` 演算子の挙動を横取り
- `deleteProperty`: `delete obj.prop` の挙動を横取り
- `apply`: 関数として呼び出された（`fn()`）ときの挙動を横取り
- `construct`: `new Fn()` でインスタンス化されたときの挙動を横取り

#### 小さなリアクティブ状態管理システムの実装

記事の締めくくりとして、`Proxy` と `Reflect` だけで最小限のリアクティブ状態システムを構築する例が示されている（Vue 3 のリアクティビティの核となる考え方と同型）。

```javascript
function reactiveState(initialState, onChange) {
  const handler = {
    get(target, property, receiver) {
      const value = Reflect.get(target, property, receiver);
      // ネストしたオブジェクトも再帰的にリアクティブ化
      if (typeof value === "object" && value !== null) {
        return reactiveState(value, onChange);
      }
      return value;
    },
    set(target, property, value, receiver) {
      const oldValue = target[property];
      const result = Reflect.set(target, property, value, receiver);
      if (oldValue !== value) {
        onChange(property, value, oldValue);
      }
      return result;
    },
  };

  return new Proxy(initialState, handler);
}

// 使用例
const state = reactiveState({ count: 0 }, (prop, newVal, oldVal) => {
  console.log(`"${prop}" が ${oldVal} → ${newVal} に変化しました`);
  render(); // 画面の再描画などをここでトリガー
});

state.count++; // "count" が 0 → 1 に変化しました
```

`set` トラップ内で変更前後の値を比較し、変化があった場合にのみコールバック（`onChange`）を発火する。これにより、状態オブジェクトへの通常のプロパティ代入（`state.count++`）だけで、副作用（再描画など）を自動的にトリガーする仕組みが作れる。

### ③ なぜ効果的か

- `Reflect` を使うことで、トラップ内のカスタムロジック以外の部分は常に「本来の正しい動作」を保証でき、継承チェーンや `receiver` 絡みのバグを防げる
- `get`/`set`/`has`/`deleteProperty`/`apply`/`construct` などの内部操作を横取りできるため、バリデーション・ロギング・アクセス制御・計算プロパティなどをオブジェクトの利用側のコードを一切変更せずに透過的に追加できる
- リアクティブ状態システムの例のように、外部ライブラリなしでVueやMobXのようなリアクティビティの仕組みを最小限の実装で理解・自作できる

---

## 4. Telegram Serverless: Telegramのサーバー上でBotロジックを直接実行

**原文リンク:** [Telegram Serverless (公式ドキュメント)](https://core.telegram.org/bots/serverless)

### ① 目的・問題意識

これまでTelegram Botを作る際は、自前でサーバーを用意し（VPS、Lambda、Cloud Functionsなど）、Webhookを受けるエンドポイントを維持し続ける必要があった。`Telegram Serverless` はこの「サーバーの管理」自体を不要にし、Bot/Mini Appのバックエンドロジックを **Telegramのインフラ上で直接** 実行できるようにする新機能（ベータ）。AIコーディングエージェントによる開発を強く意識した設計になっている。

### ② 具体的な技術詳細

#### 有効化

BotFather上で操作:
```
@BotFather → 対象のBotを選択 → Serverless → ON
```
これによりそのBot専用のCLIアクセストークン、ハンドラー、ライブラリ、組み込みデータベースが有効化される。

#### デプロイ

```bash
npx tgcloud push
```

このコマンド一発で、通常必要だったサーバーのセットアップ・コンテナのパッケージング・エンドポイントのデプロイ・スケーリング設定といった一連の作業を代替する。

#### 実行モデル

- 開発者はプレーンなJavaScriptモジュールを書くだけ
- Telegram側が高速に隔離されたV8サンドボックスでそれを実行する
- Bot APIおよび組み込みデータベースのすぐ隣で実行されるため、イベント受信 → アプリケーションロジック実行 → 保存データへのクエリ → 応答、という一連の流れを外部の独立したバックエンドを経由せずに完結できる

#### AIエージェント向けの設計

新規プロジェクトを作成すると、自動的に `AGENTS.md` ファイルがスキャフォールディングされる。これはドキュメント内で「AIコーディングアシスタントのためのオリエンテーション」と説明されており、SDKリファレンスガイドも同梱される。

### ③ なぜ効果的か

- サーバー・コンテナ・スケーリング設定を一切書かずに、`npx tgcloud push` だけでBotロジックをTelegramのインフラに直接デプロイできる
- Bot APIとデータベースが物理的に隣接しているため、外部バックエンドを経由する場合に比べてレイテンシが削減される
- `AGENTS.md` の自動生成により、AIコーディングエージェントがプロジェクト構造を理解しやすくなり、人手を介さない開発サイクルに向いている

---

## 5. LiteParse 2.0: Rust製の高速・軽量PDFパーサー

**原文リンク:** [run-llama/liteparse (GitHub)](https://github.com/run-llama/liteparse) / [LlamaIndexブログ: Up to 100x Fast Parsing with LiteParse v2.0 and Rust](https://www.llamaindex.ai/blog/liteparse-v2-0-runs-everywhere)

### ① 目的・問題意識

PDFやOffice文書からテキストを抽出する処理は、クラウドAPI（有料・レイテンシあり）に頼るか、Pythonベースの重いライブラリに依存することが多かった。`LiteParse` はLlamaIndexチームが開発したRust製のドキュメントパーサーで、クラウドAPIや独自技術に依存せず、テキスト抽出・空間的レイアウトの再構築・OCR・スクリーンショット生成をローカルで完結させる。v2.0でRustによる書き直しが行われ、以前のNodeベース実装比で大幅に高速化した。

### ② 具体的な技術詳細

#### インストール

```bash
# Node.js / TypeScript（napi-rsによるネイティブバインディング）
npm i -g @llamaindex/liteparse

# Python
pip install liteparse

# Rust
cargo install liteparse        # CLIのみ
cargo add liteparse            # ライブラリとして利用

# ブラウザ（WASM）
npm i @llamaindex/liteparse-wasm
```

WASM版以外は共通の `lit` というCLIコマンドを提供する。

#### 基本的なパース

```bash
# 通常のテキスト抽出
lit parse document.pdf

# Markdown形式に変換（構造を保持）して出力
lit parse document.pdf --format markdown -o output.md

# 空間座標付きJSONで出力
lit parse document.pdf --format json -o output.json

# 特定ページのみを対象にする
lit parse document.pdf --target-pages "1-5,10,15-20"

# OCRを無効化(テキストレイヤーがある場合など高速化)
lit parse document.pdf --no-ocr

# リモートファイルをパイプで渡す
curl -sL https://example.com/report.pdf | lit parse -
```

#### Markdown再構築時の画像モード

```bash
lit parse document.pdf --format markdown --image-mode embed \
  --image-output-dir ./images
```

- `placeholder`（デフォルト）: `![](image_pN_K.png)` のような参照のみ出力
- `off`: 画像を完全に除去
- `embed`: PNGファイルを実際にディスクへ書き出し、ローカル参照にする

#### 事前の複雑度判定（OCRが必要かどうかの事前チェック）

```bash
# ページごとの詳細な複雑度分析
lit is-complex document.pdf

# シェルスクリプトの条件式として使えるモード
lit is-complex document.pdf --quiet && \
  lit parse document.pdf --no-ocr
```
判定結果は `scanned`（スキャン画像）、`no-text`（テキストなし）、`sparse-text`（テキストが疎）、`embedded-images`（画像埋め込み）、`garbled`（文字化け）、`vector-text`（ベクターテキスト）などの理由付きで返る。

#### バッチ処理・スクリーンショット生成

```bash
# ディレクトリ単位の一括処理
lit batch-parse ./input-directory ./output-directory

# 全ページのスクリーンショット
lit screenshot document.pdf -o ./screenshots

# 特定ページを指定DPIでスクリーンショット
lit screenshot document.pdf --target-pages "1,3,5" \
  --dpi 300 -o ./screenshots
```

#### アーキテクチャ

- Rustコア + PDFium（FFI経由、独自フォーク・ビルド）でPDFテキスト抽出
- OCRはデフォルトで `tesseract-rs` をバンドル（HTTP経由でEasyOCR/PaddleOCR等の外部OCRサーバーも指定可能）
- "Grid Projection" と呼ばれる手法で、テキストの位置情報から空間的なレイアウト（見出し・表・リスト・画像・リンク）を再構築
- `.docx` / `.pptx` / `.xlsx` は LibreOffice、画像形式（jpg/png/gif/bmp/tiff/webp）は ImageMagick 経由で変換
- OCRはCPUコア数−1のワーカーで並列実行

### ③ なぜ効果的か

- クラウドAPIに一切依存せず、ローカル・オフラインで完結するためコストとレイテンシを削減できる
- Node/Python/Rust/WASM全てで同一のRustコアを共有するため、言語をまたいでも挙動が一貫する
- `is-complex` による事前判定で、OCRが不要な文書には重い処理をスキップでき、バッチ処理全体のコストを最適化できる
- v2.0のRust化により、小規模文書で5〜100倍、大規模文書でも約3倍の高速化を実現

---

## 6. actions/setup-node@v7: GitHub ActionsでのNode.jsセットアップが更新

**原文リンク:** [actions/setup-node Releases](https://github.com/actions/setup-node/releases) ([v7.0.0](https://github.com/actions/setup-node/compare/v6...v7.0.0))

### ① 目的・問題意識

GitHub Actionsワークフローの多くが利用する `actions/setup-node` の内部実装が、npmのサプライチェーン攻撃（キャッシュポイズニング等）や認証方式の変化に対応する形でアップデートされた。v6.4.0からv7.0.0への更新で、内部実装のモダン化とセキュリティ強化が図られている。

### ② 具体的な技術詳細

#### 主な変更点

- **ESMへの完全移行**: アクション内部を最新の `@actions/*` パッケージ群との互換性のためESM (ECMAScript Modules) 化。アクションの入力・出力・挙動自体に変更はない
- **新しい出力の追加**: `cache-primary-key` と `cache-matched-key` が新たに出力として追加され、キャッシュの挙動をワークフロー側から追跡できるようになった
- **`always-auth` 入力の削除**: 非推奨だった `always-auth` オプションを削除。npmの将来のリリースでもサポートされなくなるため
- **`NODE_AUTH_TOKEN` のダミーexport除去**、`getManifest` での `mirrorToken` の扱いを明示的に指定された場合のみ使うよう修正
- **`@actions/cache` を5.1.0にアップグレード**し、キャッシュ書き込みが拒否された場合のログが改善

#### ドキュメントの追加

- npmへの publish を **Trusted Publisher (OIDC)** 認証で行うためのガイド
- **キャッシュポイズニング対策**に関する推奨設定の更新
- リストアオンリー（restore-only）キャッシュの挙動の明確化

#### 利用例

```yaml
- uses: actions/setup-node@v7
  with:
    node-version: '24'
    cache: 'npm'

- run: npm ci
- run: npm test
```

### ③ なぜ効果的か

- `cache-primary-key` / `cache-matched-key` の出力追加により、キャッシュヒット率のデバッグやCI高速化のチューニングがワークフローYAML上で直接可能になる
- OIDCベースのTrusted Publisher対応により、npmトークンをGitHub Secretsに保存せずにpublishできるようになり、トークン漏洩のリスクを構造的に削減できる
- ESM移行は利用者からは非表示の内部変更だが、依存パッケージ（`@actions/cache` 等）の最新化に追従しやすくなり、今後のセキュリティパッチ適用が早くなる

---

## 7. Mediabunny: JavaScriptだけで完結するメディア処理ツールキット

**原文リンク:** [Vanilagy/mediabunny (GitHub)](https://github.com/Vanilagy/mediabunny) / [mediabunny.dev](https://mediabunny.dev/)

### ① 目的・問題意識

動画・音声のブラウザ内での読み込み・書き込み・変換（transmux/transcode）には、従来FFmpeg.wasmのような重量級ライブラリが必要だった。`Mediabunny` は依存ゼロの純粋なTypeScriptで実装され、WebCodecs APIを介したハードウェアアクセラレーションを活用しつつ、ツリーシェイキング可能（使う機能だけをバンドルに含められる）な設計により、gzip後5KBからという軽量なバンドルサイズを実現する。

### ② 具体的な技術詳細

#### インストール

```bash
npm install mediabunny
```
スクリプトタグでの読み込みにも対応（グローバル `Mediabunny` オブジェクト）。ES2021+、TypeScript 5.7+が必要。

#### メタデータの読み取り

```typescript
import { Input, ALL_FORMATS, BlobSource } from 'mediabunny';

const input = new Input({
    source: new BlobSource(file),
    formats: ALL_FORMATS,
});

const duration = await input.computeDuration(); // 秒単位
const videoTrack = await input.getPrimaryVideoTrack();
const displayWidth = await videoTrack.getDisplayWidth();
const displayHeight = await videoTrack.getDisplayHeight();
const rotation = await videoTrack.getRotation();

const audioTrack = await input.getPrimaryAudioTrack();
const sampleRate = await audioTrack.getSampleRate();
const channels = await audioTrack.getNumberOfChannels();

const { title, artist, album } = await input.getMetadataTags();
```

#### メディアファイルの生成（Canvasから動画を書き出す）

```typescript
import { Output, Mp4OutputFormat, BufferTarget, CanvasSource, QUALITY_HIGH } from 'mediabunny';

const output = new Output({
    format: new Mp4OutputFormat(),
    target: new BufferTarget(),
});

const videoSource = new CanvasSource(canvas, {
    codec: 'avc',
    bitrate: QUALITY_HIGH,
});
output.addVideoTrack(videoSource);

await output.start();
// ここでフレームを追加していく
await output.finalize();

const buffer = output.target.buffer; // 完成したMP4ファイル
```

#### フォーマット変換

```typescript
import { Input, Output, Conversion, ALL_FORMATS, BlobSource, WebMOutputFormat } from 'mediabunny';

const input = new Input({
    source: new BlobSource(file),
    formats: ALL_FORMATS,
});

const output = new Output({
    format: new WebMOutputFormat(),
    target: new BufferTarget(),
});

const conversion = await Conversion.init({ input, output });
await conversion.execute();
```

#### 新機能: Apple ProRes / HLS対応

- ISOBMFF・Matroska両方でApple ProResの読み書きに対応（コーデック文字列 `'prores'`）
- 拡張パッケージ `@mediabunny/prores` により、TurboResベースの超高速ProResデコーダーを追加（全てのProResフレーバーに対応）
- HLSの読み書き対応はPRとして進行中

#### 対応フォーマット・コーデック

MP4, MOV, WebM, MKV, HLS, WAVE, MP3, Ogg, ADTS, FLAC, MPEG-TS のコンテナに加え、25種類以上のビデオ/オーディオ/字幕コーデックをサポート。リサイズ・回転・クロップ・リサンプリング・トリミングなどの処理も可能。

### ③ なぜ効果的か

- 依存ゼロ・ツリーシェイキング対応により、実際に使う機能だけをバンドルでき、5KB gzipからという軽量な導入が可能
- WebCodecs APIを直接ラップすることで、ソフトウェアデコード専用のFFmpeg.wasm系ライブラリよりハードウェアアクセラレーションの恩恵を受けやすい
- サーバーサイド（Node/Bun/Deno）とブラウザの両方で同じAPIが使えるため、クライアント・サーバーでコードを共有できる

---

## 8. Tinbase: 単一バイナリで動くSupabase互換バックエンド

**原文リンク:** [sanketsahu/tinbase (GitHub)](https://github.com/sanketsahu/tinbase) / [tinbase.dev](https://www.tinbase.dev/)

### ① 目的・問題意識

Supabaseをローカルで動かす公式CLIはDockerに依存しており、開発環境のセットアップコストが高い。`Tinbase` はDocker不要・単一バイナリ（約58MB）で、本物のPostgres（RLS: Row Level Security込み）またはWASM版Postgresを起動し、`supabase-js` を無改造で使える開発/プロトタイピング用バックエンド。

### ② 具体的な技術詳細

#### クイックスタート

```bash
npx tinbase start
```
起動すると以下が出力される:
```
API URL: http://127.0.0.1:54321
anon key: eyJ...
service_role key: eyJ...
```

#### エンジンの選択（`--engine` フラグ）

```bash
# ネイティブエンジン（macOS/Linuxのデフォルト）: 組み込みPostgres 17、起動時約59MB RAM
npx tinbase start --engine native

# WASMエンジン（Windowsのデフォルト、ブラウザ対応）: PGliteをWASMにコンパイル
npx tinbase start --engine wasm

# pg-memエンジン: 純JS製の超軽量インメモリエンジン（ブラウザプレビュー向け）
npx tinbase start --engine pgmem
```

#### `supabase-js` との連携（無改造で動く）

```javascript
import { createClient } from '@supabase/supabase-js'

const supabase = createClient('http://127.0.0.1:54321', ANON_KEY)

// 通常のSupabase操作がそのまま動く
await supabase.auth.signUp({ email: 'me@example.com', password: 'secret123' })
await supabase.from('todos').insert({ title: 'hello' })

const { data } = await supabase
  .from('todos')
  .select('*, author:users(name)')
  .eq('done', false)

// Realtimeサブスクリプションも無改造で動作
supabase
  .channel('feed')
  .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'todos' }, console.log)
  .subscribe()
```

REST・Auth・Storage・Realtimeが本家Supabaseと同じワイヤープロトコルを話すため、公式SDKをそのまま向けられる。また `supabase/migrations/*.sql` と `seed.sql` を公式CLIと同じ形式で読み込めるため、Tinbaseで使ったマイグレーションファイルをそのままホスティング版Supabaseにも流用できる。

### ③ なぜ効果的か

- Docker・Node・npmすら不要（単一58MB実行ファイル）で、CI環境や軽量なローカル開発環境の構築が一瞬で終わる
- `supabase-js` が無改造で動くため、開発時はTinbase、本番はホスティングSupabaseという切り替えが設定変更だけで済む
- WASM/pg-memエンジンを選べることで、ブラウザ内プレビューやサンドボックス環境など、ネイティブプロセスを起動できない制約のある環境にも対応できる

---

## 9. @neon/sdk: NeonのTypeScript向け新クライアント

**原文リンク:** [Introducing @neon/sdk — Neon Blog](https://neon.com/blog/neon-sdk)
**執筆:** Andre Landgraf

### ① 目的・問題意識

旧来の `@neondatabase/api-client` はOpenAPIスペックからの自動生成コードがそのまま公開されており、使い勝手（型・エルゴノミクス）に難があった。`@neon/sdk` はゼロ依存・fetchベース・ESM専用で書き直され、OpenAPIスペックから生成しつつ、その上に使いやすいレイヤーを重ねた新クライアント。Neon Platform API全体（プロジェクト・ブランチ・データベース・Object Storage・Functions・AI Gateway）をカバーする。

### ② 具体的な技術詳細

#### インストール

```bash
npm install @neon/sdk
```

#### `createAndConnect` による1コールでのプロジェクト作成〜接続

```typescript
import { createNeonClient } from "@neon/sdk";

const neon = createNeonClient({
  apiKey: process.env.NEON_API_KEY!,
});

const { data, error } = await neon.projects.createAndConnect({
  name: "my-app",
});

if (error) throw error;
const { project, connectionString } = data;
```

`createAndConnect` は、プロジェクトのプロビジョニング → プロビジョニング操作の完了待ち（ポーリング） → 接続文字列の取得、という一連の非同期処理を1回の呼び出しに集約している。内部でポーリングを隠蔽しているため、呼び出し側は「リソースが実際に使える状態になってから」resultを受け取れる。

#### カバー範囲

- プロジェクト・ブランチ・データベースのCRUD操作
- Object Storage、Functions、AI Gatewayといったバックエンドサービスの管理

### ③ なぜ効果的か

- fetchベース・ゼロ依存のため、バンドルサイズや依存関係の脆弱性リスクを抑えられる
- `createAndConnect` のようなエルゴノミックなラッパーにより、「プロジェクト作成→プロビジョニング完了待ち→接続情報取得」という定型処理をボイラープレートなしで書ける
- `error` を戻り値として明示的に返す設計（例外ベースでない）により、呼び出し側でのエラーハンドリングが型安全かつ明示的になる

---

## 10. Ant: 9MBに収まる自作エンジンのJavaScriptランタイム

**原文リンク:** [themackabu/ant (GitHub)](https://github.com/themackabu/ant) / [antjs.org](https://antjs.org/)

### ① 目的・問題意識

Node.js や Bun はランタイム単体で50〜100MBに達することがあり、CLIツールや小さなDockerコンテナ、サーバーレス関数、エッジ環境に「同梱」するには重すぎる。`Ant` は、V8/JSC/SpiderMonkeyのラッパーではなく **完全に自作したJavaScriptエンジン「Ant Silver」** を採用し、8MB（`-Os` 最適化ビルドでは4.1MB）というバイナリサイズと、5ms台のコールドスタートを実現する。

### ② 具体的な技術詳細

#### インストール

```bash
curl -fsSL https://antjs.org/install | bash
```

#### バイナリサイズ

| ビルド | サイズ |
|---|---|
| 標準ビルド | 約8MB |
| `-Os` 最適化ビルド | 約4.1MB |

#### コールドスタート比較（Honoをimportしてルートを登録し終了するまでの計測、M4 Pro環境）

| ランタイム | 平均時間 | 相対速度 |
|---|---|---|
| **Ant** | **5.5ms** | 1.00x |
| Bun | 10.6ms | 1.93倍遅い |
| Deno | 24.8ms | 4.51倍遅い |
| Node | 28.7ms | 5.22倍遅い |

#### アーキテクチャ

- 独自エンジン「**Ant Silver**」をゼロから実装（C言語がコードベースの74%を占める）
- JITコンパイラには **MIRのフォーク**（「ほぼコンパイル済みに近い性能を実現する軽量バックエンド」）を採用
- Node互換API、VM分離されたサンドボックス、Wasmサポートを標準搭載

#### 仕様準拠状況

| テストスイート | 準拠率 | 詳細 |
|---|---|---|
| compat-table | **100%** | 1511/1511（ES1〜ES5, ES6, ES2016+, ESNext） |
| test262 | 約64% | 実運用でよく使われる範囲を優先して改善中 |

WinterTC の Minimum Common API（サーバーサイドJS runtime間の相互運用のための標準）をターゲットにしている。

### ③ なぜ効果的か

- V8などをラップせず自前実装のため、不要な機能を削ぎ落として8MB/4.1MBという極小サイズを実現できている
- コールドスタートがNodeの約1/5、Bunの約半分であるため、サーバーレス関数のコールドスタート遅延やCLIツールの起動待ちを大幅に削減できる
- Node互換APIを提供しつつサンドボックス分離も備えるため、信頼できないコードの実行環境（プラグインシステムなど）としても使える

---

## 11. Git Worktreesへの優しい入門

**原文リンク:** [A gentle introduction to Git worktrees — Human Who Codes](https://humanwhocodes.com/blog/2026/07/introduction-git-worktrees/)
**執筆:** Nicholas C. Zakas

### ① 目的・問題意識

Git worktree自体は10年以上前からある機能だが、多くの開発者に知られていない、または誤解されている。1つのリポジトリに対して複数のブランチを別々のディレクトリに同時にチェックアウトできるこの機能は、通常の並行開発だけでなく、複数のAIコーディングエージェントを同時に異なるブランチで走らせたい場合の「並行化問題」に対するエレガントな解決策になる。本記事はその使い方を分かりやすく解説する。

### ② 具体的な技術詳細

記事では基本コマンド（`git worktree add`、`git worktree list`、`git worktree remove` など）を用いて、以下のようなワークフローを解説している。

- メインの作業ディレクトリはそのままに、`feature-branch` 用の別ディレクトリを追加で用意する
- 各worktreeは独立したファイルシステム上のディレクトリを持つため、`git checkout` によるブランチ切り替え時に発生する「ビルド成果物の再生成」「node_modulesの状態変化」といった問題が起きない
- 複数のAIコーディングエージェント（あるいは複数の作業ストリーム）を、それぞれ専用のworktreeディレクトリに割り当てることで、同一リポジトリに対して衝突なく並行作業できる

### ③ なぜ効果的か

- ブランチ切り替えのたびにコンパイル済み成果物や依存関係を再構築するコストがなくなる
- 1つのリポジトリのクローンを複数持たずに済むため、ディスク使用量やfetch/pullの手間を削減できる
- 複数のAIエージェントやターミナルセッションを並行して走らせる際に、それぞれが独立したディレクトリで作業できるため、ブランチの取り違えやファイルの競合を構造的に防げる

---

## 12. GitHub Actions上のPlaywrightを本当に高速化するセットアップ

**原文リンク:** [Playwright on GitHub Actions: The setup that actually runs fast — Endform](https://endform.dev/blog/playwright-github-actions)
**執筆:** Jakob Norlin

### ① 目的・問題意識

GitHub Actions上でPlaywrightのテストを走らせると、ジョブが3〜4分アイドル状態になった後、テストが1つずつ順番にしか実行されないという問題によく遭遇する。これは「テストスイート自体の問題」ではなく「設定の問題」であり、原因は主に2つ: (1) ブラウザバイナリのインストールに毎回30〜90秒・300〜500MBを費やしている、(2) 並列度（parallelism）が活かされていない。

### ② 具体的な技術詳細

#### 問題1: ブラウザバイナリの再インストール

Playwrightのデフォルトのワークフローでは、実行のたびにブラウザバイナリ一式（300〜500MB）をダウンロードし直しており、これはコードがほぼ変化しない中で毎回のプッシュで繰り返される無駄な待ち時間になっている。対策として `actions/cache` を使い、キャッシュキーに **ランナーのOS** と **インストールされているPlaywrightパッケージの正確なバージョン** を含めることで、バージョンが変わらない限りキャッシュがヒットし続けるようにする。

#### 問題2: 並列度とシャーディングのトレードオフ

マトリクス戦略で複数のジョブを並列実行する場合、同じキャッシュキーを持つジョブが複数同時に走ると、GitHubのキャッシュの「不変（immutable）」仕様上、既存のエントリは上書きされないものの、複数ジョブがそれぞれインストール＆保存を試みて時間を浪費してしまう。これに対しては次のような選択肢がある。

- 準備専用のジョブで1回だけインストール・保存を行い、他のシャードはそれを待つ（ただしこれがクリティカルパスを伸ばす可能性がある）
- 最初の通常のワークフロー実行でキャッシュを埋めさせ、パッケージアップグレード直後の重複コストは許容する

また記事は「シャーディングは万能ではない」とも指摘する。シャーディングはテストスイートを複数の並列CIジョブに分割する強力な手法だが、ほとんどのチームがまだ抱えていない問題を解決するためのものであり、テスト数がそれほど多くないスイートであれば、ブラウザキャッシュと並列度のチューニングだけで、シャーディングなしに単一ランナー上で5分未満に収められるとしている。

### ③ なぜ効果的か

- ブラウザバイナリのキャッシュにより、変化しない300〜500MBのダウンロードを毎回省略でき、CI全体の待ち時間を大幅短縮できる
- キャッシュキーにOSとPlaywrightバージョンを含めることで、バージョンアップ時には自動的に新しいキャッシュが作られ、古いバイナリを誤って使い続けるリスクを避けられる
- シャーディングという「重い解決策」に飛びつく前に、キャッシュと並列度という「設定変更だけで終わる」対策を優先することで、CIパイプラインの複雑化を避けられる

---

## 13. GitHubメンテナーが今週有効化すべき6つのセキュリティ設定

**原文リンク:** [6 security settings every GitHub maintainer should enable this week — The GitHub Blog](https://github.blog/security/6-security-settings-every-github-maintainer-should-enable-this-week/)
**執筆:** Joseph Katsioloudes

### ① 目的・問題意識

OSSプロジェクトのメンテナーは、脆弱性報告の受け口や依存関係の管理、コードレベルの静的解析など、複数のセキュリティ機能を有効化していないケースが多い。本記事はGitHubが無料で提供する設定のうち、今すぐ有効化すべき6つを具体的に紹介する。

### ② 具体的な技術詳細（6つの設定）

1. **`SECURITY.md` ファイルの設置**: 脆弱性を発見した人がどこに（メールアドレスなど）報告すればよいかを明示し、公開の場に投稿される前に直接連絡してもらえるようにする
2. **Private Vulnerability Reporting (PVR)**: 研究者がリポジトリに対して非公開のセキュリティアドバイザリを提出できる機能。メンテナーは公開の目に触れる前にトリアージし、開示タイミングを自分でコントロールできる
3. **Dependabot と Dependency Review**: 依存パッケージの既知脆弱性を自動検知し、PRに追加/更新される依存関係の可視性を提供する
4. **Code Scanning（CodeQL）**: SQLインジェクションやコマンドインジェクションといった典型的なバグパターンの静的解析に加え、安全でないGitHub Actionsワークフローの検出も行う。OSSプロジェクトには無料
5. **Secret Scanning（Push Protection込み）**: シークレットのコミットを検知し、特にプッシュ時点でブロックする Push Protection の有効化が重要
6. **ブランチ保護 (Branch Protection)**: デフォルトブランチに対し、最低1件のレビュー承認を必須とするPRベースのワークフローを強制し、レビューを経ない不正/事故によるマージを防ぐ基礎的な対策

### ③ なぜ効果的か

- PVRとSECURITY.mdの組み合わせにより、脆弱性が公開の場でゼロデイとして晒される前に、メンテナーがコントロールされた形で対応できる
- Dependabot・Code Scanning・Secret Scanningはいずれも無料かつ自動化されており、継続的な人的コストなしにサプライチェーン攻撃・脆弱な依存・シークレット漏洩のリスクを構造的に下げられる
- ブランチ保護は最も基本的な設定でありながら、単独のコミット権限者が誤って/悪意を持ってmainブランチを壊すリスクを防ぐ最終防衛線として機能する

---

## 14. Express から Fiber へ: Go製Webフレームワークとの比較

**原文リンク:** [gofiber/fiber (GitHub)](https://github.com/gofiber/fiber) ほか比較記事

### ① 目的・問題意識

Node.jsのExpressに慣れた開発者が、パフォーマンス要求の高いバックエンドをGoで書き直したい場合の選択肢として、Express風のAPIを持つGoのWebフレームワーク `Fiber` が比較される。同じような書き味を保ちながら、Goのパフォーマンス特性を得られるかどうかが論点。

### ② 具体的な技術詳細

- FiberはExpressにインスパイアされたAPIを持ち、`fasthttp`（Goの標準 `net/http` より高速なHTTPエンジン）の上に構築されている
- ベンチマーク例: Fiberは約1,146,667 req/sec（平均レイテンシ0.4ms）、Expressは約244,847 req/sec（平均レイテンシ1.1ms）
- `fasthttp` 自体が `net/http` の約10倍の速度とされる
- 言語特性の違い: Fiberは静的型付け・コンパイル言語であるGoの並行処理機構（goroutine）を活用し、軽量なゴルーチンで大量の同時接続を効率的に処理する
- 注意点: Fiberは `fasthttp` ベースであるため、通常の `http.Handler` をラップする標準Goミドルウェアはそのままでは動作せず、Fiber専用のミドルウェアまたはアダプタが必要になる

### ③ なぜ効果的か

- Node/Expressの開発者にとって学習コストの低いAPI形状のまま、Goのランタイム性能（並行処理・コンパイル言語としての実行速度）を得られる
- goroutineベースの並行モデルにより、数千の同時接続を低メモリオーバーヘッドで捌ける
- 一方でミドルウェアエコシステムがExpress/Node向けと分断されるため、既存のExpressミドルウェア資産をそのまま持ち込めない点はトレードオフとして意識する必要がある

---

## 15. Node.js 20 が Vercel の Builds/Functions で 2026年10月1日に非推奨に

**原文リンク:** [Node.js 20 is being deprecated on October 1, 2026 — Vercel Changelog](https://vercel.com/changelog/node-js-20-is-being-deprecated)

### ① 目的・問題意識

Node.js 20は2026年4月30日に本家のEOL（End of Life）を迎えており、Vercelもこれに合わせてプラットフォーム側のNode.js 20サポートを2026年10月1日に終了する。既存デプロイへの影響と移行方法を明確にする告知記事。

### ② 具体的な技術詳細

- **既存デプロイへの影響**: 既にデプロイ済みのServerless Functionsは影響を受けず、既存関数の呼び出しは通常通り動作し続ける。影響を受けるのは**新規デプロイ**のみ
- **2026年10月1日以降の挙動**: プロジェクト設定でNode.js 20が無効化され、Functions用バージョンとして20を指定しているプロジェクトが新規デプロイを作成しようとするとエラーが表示される
- **移行方法**: プロジェクト設定、または `package.json` の `engines` フィールドでNode.jsバージョンを指定できる
- **回避策**: 期限内にアップグレードが完了できない場合、コンテナイメージとしてデプロイする方法がある。コンテナデプロイはプロジェクト設定のNode.jsバージョンの制約を受けないため、このEOL/非推奨化の対象外となる

### ③ なぜ効果的か

- 既存デプロイが即座に壊れるわけではなく、新規デプロイのみが対象という段階的な移行設計により、影響範囲を予測しやすい
- `package.json` の `engines` フィールドという既存の標準的な仕組みでバージョン指定ができるため、移行作業自体は最小限で済む
- コンテナデプロイという「常に非推奨化の対象外になる」エスケープハッチが用意されているため、どうしても移行が間に合わない場合の実務的な逃げ道がある

---

## リリースラウンドアップ（簡易まとめ）

| パッケージ | バージョン | 概要 |
|---|---|---|
| [Nano ID](https://nodeweekly.com/link/188003/rss) | 6.0 | URLセーフな一意文字列IDライブラリ。4倍高速化 |
| [c8](https://nodeweekly.com/link/188004/rss) | 12.0 | Node.js組み込みのV8カバレッジデータを使ったIstanbul互換カバレッジレポートツール |
| [np](https://nodeweekly.com/link/188006/rss) | 12.0 | npm publishをより安全にするツール。npm 12とステージド publish に対応 |
| [HyperExpress](https://nodeweekly.com/link/188007/rss) | 7.0 | uWebSockets.js上に構築された高性能Nodeサーバー |
| [Helmet](https://nodeweekly.com/link/188008/rss) | 8.3 | Express向けHTTPセキュリティヘッダー強化ミドルウェア |
| [node-auth0](https://nodeweekly.com/link/188009/rss) | 6.0 | Auth0公式Nodeライブラリのメジャーアップデート |
| [ESLint](https://nodeweekly.com/link/188011/rss) | 10.7 | JavaScript/TypeScriptの静的解析ツール |

## エコシステム動向（簡易まとめ）

| トピック | 概要 |
|---|---|
| [AWS SDK for JavaScript v3](https://nodeweekly.com/link/188012/rss) | TypeScriptバージョンサポート方針に変更 |
| [GitHub Dependabot](https://nodeweekly.com/link/188013/rss) | バージョン更新PRにデフォルトで72時間のクールダウン期間を導入（セキュリティ修正は対象外） |
| [TC39 第115回会合](https://nodeweekly.com/link/188014/rss) | Await Dictionary、Decimal、Import Defer、Map.take などの提案が議題に |
| [Cloudflare Drop](https://nodeweekly.com/link/188018/rss) | フォルダやZIPをドラッグ&ドロップするだけでサイトを即時デプロイできる新機能 |
| [plx Transpiler](https://nodeweekly.com/link/188020/rss) | JavaScript, PHP, Python, Ruby, COBOLをPostgresのPL/pgSQLに変換するトランスパイラ |
| [NodeConf EU 2026](https://nodeweekly.com/link/187980/rss) | 2026年9月29〜30日、イタリアで開催 |

---

## まとめ

| トピック | 重要度 | アクション |
|---|---|---|
| 1本のHTTPリクエストの~200ms可視化 | ★★★ | チーム内のオンボーディング教材として活用する |
| Fetch APIのエラーコード提案 | ★★☆ | 標準化動向を注視し、リトライロジック設計時に参考にする |
| Proxy / Reflect 深掘り | ★★★ | 自作リアクティブシステムやバリデーション層の実装に応用する |
| Telegram Serverless | ★★☆ | 軽量なBot/Mini App開発の選択肢として検討する |
| LiteParse 2.0 | ★★★ | クラウドAPI不要のPDF/文書処理基盤として本番導入を検討する |
| actions/setup-node@v7 | ★★★ | 速やかにv7へアップグレードし、OIDC publishへの移行を検討する |
| Mediabunny | ★★☆ | ブラウザ内メディア変換が必要な場面でFFmpeg.wasmの代替として検討する |
| Tinbase | ★★☆ | Docker無しのSupabaseローカル開発環境として試す |
| @neon/sdk | ★★☆ | Neon APIをコードから操作する際に `@neondatabase/api-client` から移行する |
| Ant ランタイム | ★★☆ | サーバーレス/エッジ/CLIでのコールドスタート改善に検証してみる |
| Git Worktrees入門 | ★★★ | 複数AIエージェントを並行稼働させる開発フローに即導入する |
| Playwright on GitHub Actions高速化 | ★★★ | ブラウザバイナリキャッシュのキー設定を今すぐ見直す |
| GitHubの6セキュリティ設定 | ★★★ | 自分のOSSリポジトリで未設定のものがないか今すぐチェックする |
| Express vs Fiber | ★★☆ | 高スループットが必要なサービスの再実装候補として比較検討する |
| Node.js 20 のVercel非推奨化 | ★★★ | 2026年10月1日までにNode.js 22/24への移行を完了させる |
