# Node Weekly Issue #641 まとめ (September 17, 2026)

> 出典: [Node Weekly Issue #641: Node 26.9 turns on FFI by default](https://nodeweekly.com/issues/641)
> （本家サイトは直接アクセスが 403 で拒否されるため、daily.dev のミラーおよび各記事の一次情報源から内容を再構成）

---

## 1. Node.js 26.9.0: `node:ffi` がデフォルトで有効化

**原文リンク:** [Node.js — v26.9.0 (Current) Release Blog](https://nodejs.org/en/blog/release/v26.9.0) / [Node.js FFI ドキュメント](https://github.com/nodejs/node/blob/main/doc/api/ffi.md) / [Node.js 26.9 turns node:ffi on by default at 37 nanoseconds a call — dev.to](https://dev.to/alexgeorgiev17/nodejs-269-turns-nodeffi-on-by-default-at-37-nanoseconds-a-call-ngj)

### ① 目的・問題意識

これまで Node.js でネイティブライブラリ（C/C++ で書かれた `.so` / `.dll` / `.dylib`）を呼び出すには、`node-gyp` や `node-ffi-napi`、あるいは Rust 製の `napi-rs` を使ってコンパイル済みのネイティブアドオンをビルドする必要があった。これはクロスコンパイル・prebuild 配布・ABI 互換性など、配布側にも利用側にも大きな負担を強いてきた。

`node:ffi` は Node.js 26.1.0 で実験的機能として導入された「ビルド不要で動的ライブラリの関数を直接呼び出せる」モジュールで、26.9.0（2026年9月16日リリース、リリースマネージャー: Antoine du Hamel）でついに**デフォルト有効**になった。26.8.2 までは `--experimental-ffi` フラグが必要だったが、26.9.0 からはそのフラグの意味が反転し、**無効化するためのフラグ**になった。

### ② 具体的なコード・技術詳細

#### モジュールの読み込みと権限モデル

```mjs
import ffi from 'node:ffi';
```
```cjs
const ffi = require('node:ffi');
```

Node.js の Permission Model（`--permission`）を有効にしている場合は `--allow-ffi` フラグが別途必要。

#### 型システム

| カテゴリ | 型名 | 省略形 |
|---------|------|--------|
| 基本 | `void`, `char` | - |
| 整数 | `int8`/`uint8`, `int16`/`uint16`, `int32`/`uint32`, `int64`/`uint64` | `i8`/`u8`/`bool`, `i16`/`u16`, `i32`/`u32`, `i64`/`u64` |
| 浮動小数 | `float32`, `float64` | `f32`/`float`, `f64`/`double` |
| その他 | `pointer`, `string`, `buffer`, `arraybuffer`, `function` | `ptr`, `str` |

定数としても公開: `ffi.types.VOID`、`ffi.types.POINTER`、`ffi.types.INT_32` など。

#### 基本的な呼び出し方（`dlopen`）

```cjs
const { dlopen, suffix } = require('node:ffi');

const { lib, functions } = dlopen(`./mylib.${suffix}`, {
  add_i32: { arguments: ['int32', 'int32'], return: 'int32' },
});

console.log(functions.add_i32(20, 22)); // 42
```

`using` 宣言（明示的リソース管理）を使った自動クローズ:

```mjs
import { dlopen, suffix } from 'node:ffi';

{
  using handle = dlopen(`./mylib.${suffix}`, {
    add_i32: { arguments: ['int32', 'int32'], return: 'int32' },
  });
  console.log(handle.functions.add_i32(20, 22));
} // スコープを抜けると自動的にライブラリを閉じる
```

#### `DynamicLibrary` クラス

```cjs
const { DynamicLibrary, suffix } = require('node:ffi');

const lib = new DynamicLibrary(`./mylib.${suffix}`);
const add = lib.getFunction('add_i32', {
  arguments: ['int32', 'int32'],
  return: 'int32',
});

console.log(add(20, 22));
console.log(add.pointer); // ネイティブ関数のアドレス（BigInt）
```

主なメソッド: `getFunction(name, signature)` / `getFunctions(definitions)`（一括解決）/ `registerCallback(signature, callback)` / `unregisterCallback(pointer)` / `refCallback()` / `unrefCallback()`。

#### JS 関数をネイティブ側から呼べるコールバック

```cjs
const callback = lib.registerCallback(
  { arguments: ['int32'], return: 'int32' },
  (value) => value * 2,
);
```

コールバックの制約:
- 作成されたスレッド上でのみ実行される
- 例外を投げたり Promise を返してはならない
- 宣言した型どおりの値を返す必要がある
- 実行中に自身が属するライブラリを close してはならない
- 実行中に自分自身を unregister してはならない

#### メモリ操作ヘルパー

```cjs
const { getInt32, getUint64, setInt32, setUint64 } = require('node:ffi');

const intValue = getInt32(pointer, 0);
const bigValue = getUint64(pointer, 8);

setInt32(pointer, 0, 42);
setUint64(pointer, 8, 999n);
```

```cjs
const { toString, exportString, toBuffer, toArrayBuffer } = require('node:ffi');

const str = toString(nativePointer);            // NUL 終端 UTF-8 文字列を読む
exportString('Hello', nativePtr, 100);           // JS 文字列をネイティブメモリへ書き込む

const buf = toBuffer(ptr, 256, true);   // true = メモリをコピー（安全）
const view = toBuffer(ptr, 256, false); // false = ゼロコピービュー（危険）
```

#### 呼び出し経路（内部実装）

ドキュメントによれば `node:ffi` は 3 段階の呼び出し戦略を持つ: **Fast API**（最適化された直接呼び出し）、**shared buffer**（引数を事前確保済みバッファへ書き込む方式）、**generic**（libffi ベースの汎用フォールバック）。dev.to の検証記事によれば、単純な整数演算関数の呼び出しは **約 37 ナノ秒/回** まで最適化されている（Fast API 経路）。

#### 安全性に関する警告（ドキュメントより抜粋）

- 解放済みメモリの読み書きを行わないこと（stale pointer は即座に破損・クラッシュの原因になる）
- ゼロコピービューはバッキングストアが解放された後に使用しないこと
- ネイティブシンボルに誤ったシグネチャを宣言しないこと（未定義動作を招く）
- `library.close()` や `unregisterCallback()` の後にコールバックポインタを呼び出さないこと

`node:ffi` はオーナーシップ追跡・境界チェック・ライフタイム管理を一切行わないため、安全性の責任はすべて開発者側にある。

#### 同時収録された 26.9.0 の他の目玉機能

- **`node:bench`**（`--experimental-bench` が必要）: `node:test` を模した組み込みベンチマークモジュール。

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
実行は `node --bench benchmark.mjs`。ヒストグラムの CBOR エクスポート/インポート、`createRunner()`、`runFile()`、`bench:plan` イベントなど、サードパーティ製ベンチマークツールが土台にできる最小限のプリミティブとして設計されている。
- **Web Workers** の実装が本格化（ref/unref 制御、組み込みスナップショットからのワーカー起動）。
- **DTLS（Datagram TLS）** の実験的サポートが実装完了（証明書検証の改善含む）。
- **VFS（仮想ファイルシステム）** が CJS/ESM の両モジュールローダーに統合され、Single Executable Application (SEA) でバンドルされたアセットを VFS としてマウント可能に。

### ③ なぜ効果的か

- ネイティブアドオンのビルド（`node-gyp`、prebuild 配布、ABI 互換性の管理）が不要になり、**C ライブラリを npm パッケージなしで直接呼べる**ようになった
- Fast API 経路により **約 37ns/call** という高速な呼び出しを実現しており、実用的なパフォーマンスで FFI が使える
- `using` 宣言や `DynamicLibrary` クラスによりリソース管理がしやすく設計されている
- 一方でメモリ安全性の保証は一切ないため、**信頼できないネイティブライブラリを呼ぶこと自体が新しい攻撃対象領域になる**点は要注意（サンドボックス環境や CI では `--no-experimental-ffi` 相当のフラグで明示的に無効化する運用も検討すべき）

---

## 2. GitHub、Copilot ランタイムを Node.js から Rust へ移植（Copilot 自身が実装）

**原文リンク:** [Migrating the GitHub Copilot runtime to Rust, using Copilot — The GitHub Blog](https://github.blog/ai-and-ml/generative-ai/migrating-the-github-copilot-runtime-to-rust-using-copilot/)

### ① 目的・問題意識

GitHub Copilot CLI / アプリ / SDK を支えるランタイムは元々 TypeScript + Node.js + V8 上に構築されていた。CLI 用途では問題なかったが、SDK として組み込む用途では次の問題が顕在化した。

- SDK の呼び出し側は毎回 **別プロセスとして Node.js を起動**し、JS ランタイムをロードし、JSON-RPC で通信する必要があった
- この構成が **クライアント1つあたり約 100MB のワーキングセットメモリ**を追加していた
- プロセス管理の複雑化、起動速度・スループット・サーバー密度の制限につながっていた

### ② 具体的な技術詳細

#### 移植戦略：インプレース・アトミック置換

各 PR が「既存の TypeScript コンポーネントを、新しい Rust コードを呼び出す薄いシムに置き換え、同じ変更の中で古い TypeScript を削除する」という単位で進められた。これにより：

- メインブランチは常にリリース可能な状態を維持
- 約14.5週間で135回のリリースを実施
- 既存の E2E テストが各 Rust 移植を即座に検証できた

#### 相互運用: napi-rs シム

Rust 側から Node.js を呼び出す部分は napi-rs プロジェクトの `napi` クレートを使用して実装された。移植が進むにつれて最終的にランタイムライブラリの公開エントリポイントに到達し、シム自体は「蒸発」していく（＝ TypeScript 側のラッパーコードが不要になっていく）設計。

#### 新しいランタイムが公開する 2 つの「扉」

- **napi ドア**: Node.js 向け（既存の Node ベースの CLI/SDK 利用者との後方互換性）
- **C ABI ドア**: わずか **19個のエクスポート関数**のみで構成され、SDK が別プロセスを起動せず**インプロセスで FFI 経由でランタイムを埋め込める**ようにする

#### 規模・コスト

- 本番稼働する **約 832,000 行の Rust コード**をゼロから作成
- 開発期間 **約14.5週間**、その大部分を **Copilot 自身が記述**（1名の開発者が仕上げを担当し、チームの他メンバーは通常の機能開発を継続）
- モデル使用量は **約1,363億トークン**、コストは**約 12万ドル**

#### 移植中に遭遇した回帰（リグレッション）

- **JavaScript の数値型の扱い**の違い（Rust の整数型 vs JS の `Number`/倍精度浮動小数点の意味論の齟齬）
- **`||` 演算子の意味論**の違い（JS の truthy/falsy 判定とその他言語の論理和の違い）
- **イベントループのブロッキング**（Node.js のシングルスレッド・イベントループ特有の前提が Rust 側の同期処理で崩れるケース）

### ③ なぜ効果的か

- プロセスを跨がず **インプロセス FFI 埋め込み**にすることで、クライアントあたり約100MBのメモリオーバーヘッドと、プロセス起動・JSON-RPC通信のレイテンシを削減
- **アトミック置換**戦略により、大規模書き換えでありながら常時リリース可能な状態を維持でき、実運用中のプロダクトで安全に移行できた
- **C ABI を19関数に絞る**ことで、Node 以外の言語（Python、Go など）からも同じランタイムを再利用できる可能性を残した
- Node.js/TypeScript は「プロトタイピングや CLI 実行環境としては優れているが、SDK 組み込み用途ではプロセスモデルがボトルネックになる」という教訓を示す事例であり、napi-rs のような Rust↔Node 相互運用技術の実運用実績として参考になる

---

## 3. Undici の `Agent` で TLS 接続を使い回すテクニック

**原文リンク:** [The Node.js Keep-Alive Trick That Halved Latency — Bhagya Rana (Medium)](https://medium.com/@bhagyarana80/the-node-js-keep-alive-trick-that-halved-latency-02264fc34bdf)

### ① 目的・問題意識

外部 API を呼ぶたびに新しい TCP 接続を張り、TLS ハンドシェイク（鍵交換）をゼロからやり直すと、特にクロスリージョン通信で無視できないレイテンシが発生する。デフォルトの `fetch()` やリクエストごとに `new Agent()` を作るようなコードでは、この「ウォームでない」ソケットのコストを毎回支払うことになる。

### ② 具体的な技術詳細

#### プロセス全体で単一の Keep-Alive Agent を使う

```javascript
import { Agent, setGlobalDispatcher } from 'undici';

const agent = new Agent({
  connections: 100,        // オリジンごとの最大コネクション数
  pipelining: 10,
  keepAliveTimeout: 30_000,     // アイドルソケットを何 ms 保持するか
  keepAliveMaxTimeout: 60_000,  // keep-alive の絶対上限
});

setGlobalDispatcher(agent);
```

- **リクエストごとに Agent を作らず、プロセス全体で1つの Agent を使い回す**のが鍵
- `keepAliveTimeout` は 30〜120 秒程度がロードバランサー配下では安全とされる
- Undici の Agent はオリジンごとに `Pool` を保持しており、あるオリジンへの最初のリクエストでコネクションを確立（HTTPS の場合は TLS へアップグレード）し、以降のリクエストは同じプールのソケットを再利用する
- TLS セッション自体の再利用（TLS session resumption）にも対応しており、フルハンドシェイクを毎回行わずに済む
- あわせて **DNS のキャッシュ**も行うことで、名前解決のオーバーヘッドも削減できる

#### 効果

「ほぼ常に1本のウォームなソケットで捌け、バースト時のみ数本（記事内の例では約7本）に増える」という接続パターンになり、TLS ハンドシェイクとDNS解決という2つの隠れたレイテンシ要因を消し去ることができる。

### ③ なぜ効果的か

- TLS ハンドシェイクは RTT を複数回消費するため、**接続の再利用だけでレイテンシがほぼ半減**するケースがある（記事タイトルの "Halved Latency" が示す通り）
- コネクションプーリングにより、同時多発的なリクエストでも新規ソケット数を抑えられ、相手サーバー・自サーバー双方のリソース消費を削減できる
- `undici` は Node.js の組み込み `fetch()` の内部実装でもあるため、この設定は Node 標準の `fetch` のパフォーマンスチューニングにも応用できる

---

## 4. Upyo — ランタイムを問わない新しいメール送信ライブラリ

**原文リンク:** [GitHub - dahlia/upyo](https://github.com/dahlia/upyo)

### ① 目的・問題意識

メール送信のエコシステムは長年 Nodemailer が事実上の標準だったが、Nodemailer は Edge Functions（Cloudflare Workers など）で動かない、プロバイダーごとに個別の SDK が必要、Deno/Bun 向けのライブラリがほぼ存在しないなど、断片化した状況にあった。Upyo はこれを解決する「Node.js / Deno / Bun / エッジ関数を横断する統一的な型安全 API」として設計されている。

### ② 具体的な技術詳細

#### 基本的な使い方

```javascript
import { createMessage } from "@upyo/core";
import { MailgunTransport } from "@upyo/mailgun";

const message = createMessage({
  from: "sender@example.com",
  to: "recipient@example.net",
  subject: "Hello from Upyo!",
  content: { text: "This is a test email." }
});

const transport = new MailgunTransport({
  apiKey: process.env.MAILGUN_KEY,
  domain: process.env.MAILGUN_DOMAIN
});

const receipt = await transport.send(message);
```

#### アーキテクチャ

各プロバイダーが共通の `Transport` インターフェースを実装する「トランスポート方式」を採用しており、`transport` オブジェクトを差し替えるだけでプロバイダーを切り替えられる。

#### 対応トランスポート（モノレポに15以上のパッケージ）

| カテゴリ | 対応先 |
|---------|--------|
| SMTP | ネイティブ SMTP サポート |
| HTTP プロバイダー | Lettermint, Maileroo, Mailgun, Mailtrap, Resend, SendGrid, Amazon SES, Plunk |
| プロトコル | JMAP (RFC 8620/8621 準拠) |
| ユーティリティ | テスト用モックトランスポート、リトライ/バックオフのデコレータ、OpenTelemetry 連携 |

- npm と JSR の両方でパッケージ配布
- `File` オブジェクトによる添付ファイルサポート
- 一貫したエラーハンドリング

### ③ なぜ効果的か

- **トランスポートを差し替えるだけ**でメールプロバイダーを乗り換えられるため、ベンダーロックインを避けられる
- Node.js だけでなく **Deno / Bun / エッジ関数でも同じコードが動く**ため、マルチランタイム環境を持つチームでのコード共通化に有効
- モックトランスポートが標準で用意されており、**外部送信なしにメール送信ロジックのテスト**が書きやすい

---

## 5. 分散システムにおけるサーキットブレーカーの実装

**原文リンク:** [How to Implement a Distributed Circuit Breaker — Gábor Koós](https://blog.gaborkoos.com/posts/2026-09-14-How-to-Implement-a-Distributed-Circuit-Breaker/)

### ① 目的・問題意識

単一プロセス内で完結するサーキットブレーカー（失敗率が閾値を超えたら呼び出しを止める）は、複数インスタンスにスケールしたサービスでは各インスタンスが独立した状態を持ってしまい、「あるインスタンスだけが遮断し、他は遮断していない」という不整合が起きる。これを解決するには、ブレーカーの状態を Redis のような共有ストアにバックさせ、フリート全体で一貫した判断をする必要がある。

### ② 具体的な技術詳細

#### サーキットブレーカーの3状態

| 状態 | 動作 |
|------|------|
| **Closed（閉）** | すべての呼び出しを依存先へ素通しし、結果（成功/失敗）を記録し続ける |
| **Open（開）** | 記録された失敗率が設定した閾値を超えると遷移。依存先の呼び出しを一切行わず、即座にエラーを返す |
| **Half-Open（半開）** | 固定のクールダウン期間の後、まだ壊れているかもしれないサービスに全トラフィックを流さないよう、一部の呼び出しだけを試験的に通す状態 |

#### 分散環境特有の課題（ABA 問題）

記事では「half-open」というラベルが**同一性を保証するものではなく、繰り返し貼られるラベルに過ぎない**という「ABA 問題」を指摘している。あるインスタンスが half-open だと判断してプローブを1つ通した直後に、別インスタンスも同じ Redis の状態を見て half-open だと判断し、二重にプローブを流してしまう、といった競合が起こりうる。ラベルを比較するだけでは「意思決定をした時点と同じ世界にまだいるか」は分からない、という点が分散サーキットブレーカー実装の本質的な難しさとして説明されている。

#### 実装アプローチ

Redis を状態ストアとするレジリエンスライブラリ（記事内で言及される `caracal` など）を用いて、失敗カウント・状態・クールダウンのタイムスタンプを共有ストアに持たせ、half-open 時の「試験呼び出しは1台のインスタンスだけが実行する」権利をアトミックな Redis 操作（`SETNX` や Lua スクリプトなど）で排他制御する設計が紹介されている。

### ③ なぜ効果的か

- 状態を共有ストアに集約することで、**フリート全体で一貫したブレーカー判断**ができ、一部のインスタンスだけが壊れたサービスへリクエストを送り続ける事態を防げる
- half-open 状態での「プローブの二重実行」を明示的に問題として扱っており、**分散ロックやアトミック操作の設計が不可欠**であることを実例で示している
- マイクロサービス構成でのカスケード障害対策として、単一プロセスの実装をそのままスケールさせるとハマる落とし穴を事前に学べる

---

## 6. npm v12 のインストールスクリプト無効化、Node.js 27 まで見送りに

**原文リンク:** [Update to npm blocks install scripts: What it means for security — ReversingLabs](https://www.reversinglabs.com/blog/npm-v12-blocks-install-scripts) / [GitHub to Disable npm Install Scripts by Default to Stop Supply Chain Attacks — The Hacker News](https://thehackernews.com/2026/06/github-to-disable-npm-install-scripts.html)

### ① 目的・問題意識

`npm install` 時に自動実行される `preinstall` / `install` / `postinstall` スクリプトは、サプライチェーン攻撃（悪意あるパッケージが依存関係経由でマルウェアを実行する）の主要な侵入経路になってきた。これに対応するため npm v12（2026年7月8日出荷）では **`allowScripts` のデフォルトが off** になり、明示的に許可しない限りインストールスクリプトが実行されなくなった。

しかし Node.js のリリースワーキンググループは、この新しいデフォルト挙動を Node.js 22 / 24 / 26 に同梱される npm へバックポートすることは**「あまりに破壊的すぎる (far too disruptive)」**と判断し、この保護機能は **Node.js 27（次期リリース）からのみ**有効になることを決定した。

### ② 具体的な技術詳細

- npm v12 では `install`/`preinstall`/`postinstall` などのライフサイクルスクリプトが依存パッケージに対してデフォルトで実行されなくなる
- 明示的に許可するには `.npmrc` や `package.json` の設定でスクリプト実行を許可するパッケージを指定する必要がある（allowlist 方式への転換）
- npm v12 は同時に **2FA バイパス用トークンの非推奨化**も開始している
- Node.js 22 / 24 / 26 の LTS/Current ラインでは、同梱される npm のバージョンは上がっても、この allowScripts のデフォルト変更だけは適用されない措置がとられる

### ③ なぜ効果的か

- インストールスクリプトのブロックは、**実行前にコードレビューなしで任意コードが実行される**というサプライチェーン攻撃の典型パターンを塞ぐ効果的な防御策
- 一方で、既存の CI パイプラインやネイティブモジュールのビルド（`node-gyp` 等）に依存するプロジェクトが多いため、**急な既定値変更は大量の破壊的影響**を生む。Node.js 側が影響範囲を見極めてから段階的に導入する判断は、エコシステム全体の安定性を優先した現実的なリスク管理と言える

---

## 7. GitHub Actions: `cache-mode` でキャッシュポイズニングを防止

**原文リンク:** [Control GitHub Actions cache access with cache-mode — GitHub Changelog](https://github.blog/changelog/2026-09-10-control-github-actions-cache-access-with-cache-mode/) / [GitHub Actions Adds cache-mode to Limit Cache Poisoning Risk — Socket](https://socket.dev/blog/github-actions-cache-mode)

### ① 目的・問題意識

GitHub Actions のキャッシュは、あるコンテキスト（例: `pull_request_target` のような低信頼イベント）で書き込まれたエントリを、別の信頼されたコンテキスト（例: `push` トリガーのワークフロー）で読み出せてしまう。攻撃者がキャッシュキーへの書き込み権限を得れば、信頼されたワークフローがそのキャッシュを読み込んだ際に、悪意あるビルド成果物や依存関係をそのワークフローの権限・シークレットで実行させられる（キャッシュポイズニング）。実際に 2024年の Ultralytics PyPI パッケージや 2026年5月の TanStack npm パッケージがこの手口で侵害された。

### ② 具体的な技術詳細

#### `cache-mode` の4つの値

| 値 | 動作 |
|----|------|
| `read` | キャッシュの復元のみ許可（保存は不可）。`pull_request_target` などの低信頼イベントのデフォルト |
| `write` | 復元・保存の両方を許可。`push` などの信頼されたイベントのデフォルト |
| `write-only` | 保存のみ許可（復元は不可） |
| `none` | キャッシュへのアクセスを一切禁止 |

#### 設定例

ワークフローレベル:
```yaml
cache-mode: read
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: npm-${{ hashFiles('**/package-lock.json') }}
```

ジョブレベル（ワークフローレベルの設定を上書き可能）:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    cache-mode: write
  test:
    runs-on: ubuntu-latest
    cache-mode: read
```

- 低信頼イベントに対して `write` や `write-only` を宣言すると、キャッシュポイズニングのリスクが高まるため **警告アノテーションが表示される**

### ③ なぜ効果的か

- **最小権限の原則**をキャッシュアクセスにも適用でき、「本当に書き込みが必要なジョブだけ」に `write` を与えられる
- ワークフロー/ジョブ単位で細かく制御できるため、既存のワークフロー構成を大きく変えずに導入しやすい
- 実際に悪用実績のある攻撃（Ultralytics、TanStack のインシデント）を踏まえた設計であり、CI/CD パイプラインのサプライチェーンセキュリティ強化に直結する

---

## 8. Express 4.22.3 — セキュリティリリース

**原文リンク:** [Release v4.22.3 · expressjs/express](https://github.com/expressjs/express/releases/tag/v4.22.3)

### ① 目的・問題意識

Express 4.x 系は現在も広く使われ続けているため、脆弱性のバックポート対応が引き続き必要。今回は `path-to-regexp` に起因する脆弱性（CVE-2026-4867）の修正が中心。

### ② 具体的な技術詳細（2026年9月14日リリース）

- `path-to-regexp` を **0.1.13** に更新し **CVE-2026-4867** に対応
- `QUERY` リクエストに対する条件付き再検証（conditional revalidation）のサポートを追加
- `qs` を **~6.16.0** に更新
- npm のステージド公開（dist-tag 付き）を CI に導入
- 4名のコントリビューターが参加、うち2名は初コントリビューション

### ③ なぜ効果的か

- 依然として大量の本番サービスが依存する Express 4 系の**既知 CVE を塞ぐ**ため、影響を受けるプロジェクトは速やかな追従が推奨される
- `qs` のマイナーバージョン追従により、クエリパース周りの上流の修正も間接的に取り込まれる

---

## 9. Aedes 1.2.0 — MQTT ブローカーの QoS2 関連脆弱性を修正

**原文リンク:** [Release 1.2.0 · moscajs/aedes](https://github.com/moscajs/aedes/releases/tag/v1.2.0)

### ① 目的・問題意識

Aedes は Node.js 製の組み込み可能な MQTT ブローカー。QoS 2（Exactly Once）メッセージ配送の実装には、受信ウィンドウの境界チェック漏れや、セッションクリア時に重複排除ストアが破棄されない不備があり、これらがサービス運用上のリスクになっていた。

### ② 具体的な技術詳細（2026年9月16日リリース）

| GHSA ID | 内容 |
|---------|------|
| `GHSA-5jvp-3p2v-5qgf` | QoS 2 の受信ウィンドウに境界チェックを追加（`fix: bound the inbound QoS 2 receive window`） |
| `GHSA-p8r9-qf8w-p73r` | クリーンセッション時に QoS 2 の重複排除（dedup）ストアを破棄するよう修正（`fix: discard the QoS 2 dedup store on clean session`） |

その他、`release-it` のスキーマバージョン更新、メトリクス無効化フラグの削除など。

### ③ なぜ効果的か

- QoS 2 の受信ウィンドウに境界がないと、悪意あるクライアントが大量の未確認メッセージでブローカーのメモリを圧迫させる DoS の温床になりうるため、**境界チェックの追加はメモリ安全性・可用性の両面で重要**
- クリーンセッション時に dedup ストアを破棄しない不備は、セッションをまたいだ状態漏れにつながるため、**MQTT のセマンティクス（クリーンセッション = 状態リセット）を正しく守る**ための修正として妥当

---

## 10. pg-boss 12.33 — `TestClock` で時間依存のテストを決定論的に

**原文リンク:** [Release 12.33.0 · timgit/pg-boss](https://github.com/timgit/pg-boss/releases/tag/12.33.0)

### ① 目的・問題意識

ジョブキューライブラリ pg-boss は、ポーリング間隔・リトライ・有効期限・スケジュールなど「時間経過」に依存するロジックが多い。従来これらをテストするには実際に `setTimeout` 相当の待ち時間を挿入する必要があり、テストが遅く不安定になりがちだった。「テストが指示したときにだけ時間が進む」ことを実現する `TestClock` が導入された。

### ② 具体的な技術詳細（スキーマバージョン 42）

#### `TestClock` の利用

```javascript
const clock = new TestClock('2026-01-01T00:00:00Z')
const boss = new PgBoss({ connectionString, clock, __test__enableSpies: true })
```

```javascript
await boss.work('q', async () => {})
const id = await boss.send('q', {}, { startAfter: 60 })
await clock.tick(60_000)
await boss.getSpy('q').waitForJobWithId(id, 'completed')
```

#### データベース側の変更

スキーマバージョン 42 で `${schema}.job_now()` という安定した SQL 関数が追加され、これまで直接呼ばれていた `now()` を置き換えることで、**JS 側とデータベース側の両方の時刻をテストクロックが統一的に制御**できるようになった。アップグレード時にテーブルの書き換えや行の変更は発生しない。

#### `doctor --fix` の拡張

テスト用のクロックオーバーライドがテストインフラの異常終了によって本番データベースに残ってしまった場合、`doctor --fix` コマンドがそれを検出・修復し、パフォーマンス劣化を防ぐ。

#### 12.33.1〜12.33.3 の追加修正

- 12.33.1: バックグラウンドタイマー処理の3件のバグ修正（健全なデプロイでも処理間隔をスキップしてしまう不具合など）
- 12.33.2: `offWork` で停止中のワーカーの完了を待つよう修正
- 12.33.3: インデックスが既に有効な再試行(re-attempt)を正しく完了させる修正

### ③ なぜ効果的か

- 時間を**注入可能な依存**として扱うことで、実時間を待たずに「60秒後のジョブ開始」のようなシナリオを**即座に・決定論的に**テストできる
- JS 層だけでなく **SQL 関数レベル**でも同じクロックを参照させることで、アプリ側とDB側の時刻がズレて再現しないテストになる問題を根本的に解決している
- `doctor --fix` によるセーフガードは、テスト用の仕組みが誤って本番に混入した場合の被害を最小化する運用面の工夫

---

## 11. jsdom 30.1.0 — AI エージェントが発見した大量のパフォーマンス改善

**原文リンク:** [Release v30.1.0 · jsdom/jsdom](https://github.com/jsdom/jsdom/releases/tag/v30.1.0)

### ① 目的・問題意識

jsdom は Node.js 上で DOM をエミュレートする、テストツール（Jest、Vitest 等）の裏側で広く使われるライブラリ。今回のリリースは、コントリビューター @scttcper が Codex（AI コーディングエージェント）を大量の改善点の発掘に活用したことに捧げられている。AI が見つけた変更は人間がレビュー・整形した上で取り込まれた。

### ② 具体的な技術詳細（2026年9月17日リリース）

#### 新機能

- `document.myForm` のように `name` 属性を持つフォーム要素へ `document` から直接アクセスできる Named Access の追加
- ストレージ容量超過や過大な `crypto.getRandomValues()` リクエストに対する `QuotaExceededError` の実装
- 要素・属性・ドキュメントタイプ作成時の「緩和された DOM 命名規則」のサポート

#### パフォーマンス改善

- DOM構築、ツリーの変更、Range 操作、ライブコレクションへのアクセス（特に大規模ドキュメントで）を大幅に最適化
- `getComputedStyle()`、スタイル変更、CSS シリアライズのパフォーマンス改善
- イベントディスパッチとフォームコントロールのルックアップの高速化
- DOM ノード・属性・リスナー・オブザーバー全般でのメモリ消費削減

#### バグ修正（抜粋）

| カテゴリ | 内容 |
|---------|------|
| セレクタ/クエリ | v30.0.0 で発生した `querySelectorAll()` の回帰を修正、CSS属性セレクタの大文字小文字問題、`querySelector()` のID一致の不具合を解消 |
| スタイル/レンダリング | style要素更新後のスタイルシート順序、ネストした `@import`/`@media` の処理、border-width の計算、`font-weight` キーワードの数値解決を修正 |
| メモリ/ライフサイクル | MutationObserver のメモリリーク解消、破棄済みドキュメントでのリソース読み込み防止 |

### ③ なぜ効果的か

- テストランナー経由で jsdom を使う膨大な数のプロジェクトにとって、**テスト実行時間の短縮**に直結する
- AI エージェントによる大規模な改善発掘という開発プロセス自体が事例として興味深く、「AIが見つけた変更を人間がPRとして整形・レビューする」というワークフローの実例になっている
- 破壊的変更を伴わない形で多数のバグ修正・最適化が入っているため、**多くのプロジェクトが低リスクでアップグレード可能**

---

## 12. XO 5.0 — ESLint ラッパーリンターの破壊的変更

**原文リンク:** [Release v5.0.0 · xojs/xo](https://github.com/xojs/xo/releases/tag/v5.0.0)

### ① 目的・問題意識

XO は「素晴らしいデフォルト設定を持つ ESLint ラッパー」として人気の JavaScript/TypeScript リンター。v5.0.0 では設定の一貫性を高めるための破壊的変更と、対応ファイル種別の拡張が行われた。

### ② 具体的な技術詳細（2026年9月16日リリース、v5.0.1 で追加パッチ）

#### 破壊的変更

1. **`space` / `semicolon` / `prettier` オプションがグローバル専用に**: `files`、`ignores`、`basePath` を持つ設定項目でこれらを設定するとエラーになる。ファイル別の上書きには `@stylistic/indent`、`@stylistic/semi`、`prettier/prettier` ルールを使う必要がある
2. `unicorn/prefer-minimal-ternary` の `checkVaryingCallee` オプションが `checkVaryingBase` にリネーム

#### 新機能

- `eslint-package-json` の導入により `package.json` 自体をリント可能に
- `space` オプションが HTML ファイルにも適用されるように
- TypeScript の import に拡張子を強制する `n/file-extension-in-import`（`import-x/extensions` を置き換え）
- `@stylistic/function-paren-newline` を無効化
- `eslint-plugin-unicorn` を v75、`eslint-node-test` を v0.4 に更新
- 新規に **unicorn 系19ルール、node-test系10ルール、n系1ルール**を追加（async iterator、Promise resolver、文字列トリム、SQLiteのインジェクション対策、DOM操作パターンなどをカバー）

#### バグ修正

- v5.0.1: JSON / CSS ファイルのリントが動作しない不具合を修正
- Windows での TypeScript ファイルマッチングの不具合を修正

### ③ なぜ効果的か

- `space`/`semicolon`/`prettier` をグローバル専用にすることで、**プロジェクト内でスタイル設定が部分的にバラつく**という混乱を防止
- `package.json` のリントや SQLite インジェクション対策ルールなど、**リンターがカバーする領域をコードスタイル以外のセキュリティ・品質面にも広げている**
- import 拡張子強制ルールは ESM/TypeScript の相互運用でよくあるハマりどころを機械的に検出できる

---

## 13. officeParser 8.0 — PDF抽出の全面書き換えと DOCX テンプレート機能

**原文リンク:** [Release v8.0.0 · harshankur/officeParser](https://github.com/harshankur/officeParser/releases/tag/v8.0.0)

### ① 目的・問題意識

officeParser は docx / pptx / xlsx / odt / odp / ods / pdf / rtf を解析し、階層的な AST（抽象構文木）へ変換する Node.js/ブラウザ両対応ライブラリ。v8.0.0 は「PDF 処理における最大の前進」と位置づけられ、単なるフラットテキスト抽出から**構造化抽出**へと転換した。

### ② 具体的な技術詳細

#### PDF抽出の全面書き換え

以前はPDFから平坦なテキストしか取れなかったが、v8.0.0 では以下のような**実構造**を持った AST を生成する。

- レベル付きの見出し（heading with levels）
- 結合セルを含むテーブル
- リスト
- 注釈（notes）
- 複数カラムの読み順（multi-column reading order）
- 文字色

これらはタグ付き PDF であればドキュメントのタグ情報から取得し、タグがなければジオメトリ（座標情報）から再構成する。

#### DOCX / ODT 生成とテンプレート機能（マイクロソフト用語でいう mail-merge）

```javascript
const filled = await renderTemplate(templateBytes, {
  data: { name: 'Ada', amount: '$42' }
});

const batch = await renderTemplate(templateBytes, {
  data: [{ name: 'Ada' }, { name: 'Alan' }]
}); // Uint8Array[] が返る（複数ドキュメントを一括生成）
```

DOCX テンプレートの `{{placeholder}}` タグをデータオブジェクトの値で置換し、書式を保ったまま新しい `.docx` を生成する。**ラン（run）をまたいで分割されたプレースホルダーでも正しく置換**され、本文・ヘッダー・フッター・注釈・コメント内のプレースホルダーもすべて対象になる。ODT 生成（`to('odt')`）はネストしたリストや結合セル、メタデータ保持にも対応。

#### パスワード保護されたドキュメントの統一対応

```javascript
const ast = await parseOffice(file, { password: 'secret' });
```

全フォーマット共通の設定でパスワード付きドキュメントを復号できる。

#### 破壊的変更

| 変更 | 内容 |
|------|------|
| Node.js バージョン要件 | `>=18` → `>=22.13` |
| `ast.toText()` | 削除。`(await ast.to('text')).value` を使う（デフォルトでは同じ内容を返し、レイアウト・注釈・画像プレースホルダーの扱いを設定可能） |
| CLI `--toText` | 削除（`--toText was removed in v8; use --to=text instead` というエラーになる） |
| PDF内画像 | BMP ではなく **PNG** で出力されるように |
| 非推奨オプション | `outputErrorToConsole`、`ocrLanguage`、`ocrConfig.autoTerminateTimeout`、`putNotesAtLast` を削除 |

多くの破壊的変更には旧挙動へ戻すフラグ、または明確な移行先が用意されている。

### ③ なぜ効果的か

- PDFから**フラットテキストではなく構造（見出し・表・リスト）が取れる**ことで、RAG（検索拡張生成）パイプラインなどでの文書チャンク化・意味理解の精度が上がる
- DOCX テンプレートのラン跨ぎプレースホルダー対応は、Word の内部データ構造（1つの単語が複数の `<w:r>` ランに分割されることがある）特有の落とし穴を踏まえた実装であり、**実務で使えるレベルのmail-merge機能**になっている
- パスワード保護ドキュメントの統一APIにより、フォーマットごとに異なる復号処理を書き分ける必要がなくなる

---

## 14. oclif 6.0 — レガシープラグインサポートを終了

**原文リンク:** [Release 6.0.0 · oclif/oclif](https://github.com/oclif/oclif/releases/tag/6.0.0)

### ① 目的・問題意識

oclif は Salesforce が開発する CLI 生成・ビルド・リリースのためのフレームワーク（Heroku CLI などにも採用）。6.0.0 では長らく残っていたレガシープラグイン形式のサポートを整理する破壊的変更が行われた。

### ② 具体的な技術詳細（2026年9月11日リリース）

- **破壊的変更**: レガシープラグインのサポートを削除（`feat!: remove support for legacy plugins`）
- 新しいプラグインアーキテクチャへの一本化が意図されている

### ③ なぜ効果的か

- 複数のプラグイン形式を同時にサポートし続けるメンテナンスコストを削減でき、**フレームワーク内部の実装をシンプルに保てる**
- 既存の oclif ベース CLI でレガシープラグインを使っているプロジェクトは、アップグレード前に移行が必要になる点に注意

---

## 15. npmcharts.com — npm パッケージのダウンロードトレンド比較ツール

**原文リンク:** [npmcharts.com](https://www.npmcharts.com/)

### ① 目的・問題意識

複数の npm パッケージのうちどれを採用すべきか判断する際、ダウンロード数の推移という定量的なシグナルが有用だが、npm 自体にはパッケージ間の比較機能がない。

### ② 具体的な技術詳細

- 複数パッケージのダウンロード数を**同一グラフ上で比較**できるWebサービス
- 日次・週次・月次の表示切り替えと、対数スケール表示に対応
- 記事への埋め込み用にチャートを**Embeddable**な形で提供
- 総ダウンロード数、期間内ダウンロード数、依存パッケージ数などの指標を扱う

### ③ なぜ効果的か

- ライブラリ選定時に「どちらが実際に使われ続けているか」を**視覚的・定量的**に確認でき、勢いを失いつつあるパッケージへの依存を避ける判断材料になる
- ブログ記事やドキュメントに埋め込み可能なため、技術選定の根拠を共有する際にも活用できる

---

## まとめ

| トピック | 重要度 | アクション |
|---------|--------|-----------|
| Node.js 26.9.0: `node:ffi` デフォルト化 | ★★★ | ネイティブライブラリ連携が必要な箇所で `node:ffi` の採用を検討し、同時に安全性（ゼロコピー・コールバック管理）の設計を見直す |
| GitHub Copilot ランタイムの Rust 移植 | ★★★ | SDK/組み込み用途でのプロセス起動コストが問題になっている場合、napi-rs によるインプロセス化のアーキテクチャパターンを参考にする |
| Undici Agent の TLS 再利用テクニック | ★★★ | 外部API呼び出しの多いサービスでは、プロセス全体で単一の `Agent`（keep-alive設定込み）を使い回すよう見直す |
| Upyo | ★★☆ | マルチランタイム（Node/Deno/Bun/エッジ）でのメール送信基盤として新規プロジェクトで検討 |
| 分散サーキットブレーカー | ★★☆ | マイクロサービス構成でのブレーカー実装がある場合、Redisバックエンドの排他制御とABA問題を意識して設計を見直す |
| npm v12 スクリプトブロックの見送り | ★★★ | npm v12 に直接依存するプロジェクトは `allowScripts` の挙動を確認し、Node 27 での既定値変更に備える |
| GitHub Actions `cache-mode` | ★★★ | `pull_request_target` 等の低信頼トリガーを使うワークフローに `cache-mode: read` 等を設定し、キャッシュポイズニング対策を導入する |
| Express 4.22.3 | ★★★ | CVE-2026-4867 対応のため速やかにアップグレード |
| Aedes 1.2.0 | ★★★ | MQTTブローカーを運用している場合はQoS2関連の脆弱性修正のため即時アップグレード |
| pg-boss 12.33 (`TestClock`) | ★★☆ | 時間依存のジョブキューロジックのテストに `TestClock` を導入し、テストの高速化・安定化を図る |
| jsdom 30.1.0 | ★★☆ | テストスイートで jsdom を使っている場合はアップグレードして実行時間短縮の恩恵を受ける |
| XO 5.0 | ★★☆ | アップグレード時に `space`/`semicolon`/`prettier` のグローバル設定への統一を確認する |
| officeParser 8.0 | ★★☆ | PDF/Officeファイルの構造化抽出やDOCXテンプレート生成が必要な場面で採用を検討（Node 22.13+ 要件に注意） |
| oclif 6.0 | ★☆☆ | レガシープラグインを使っている oclif ベースCLIは移行計画を立てる |
| npmcharts.com | ★☆☆ | ライブラリ選定時の補助ツールとしてブックマークしておく |
