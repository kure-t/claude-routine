# Node Weekly Issue 638 まとめ (August 27, 2026)

> 出典: [Node Weekly Issue 638](https://nodeweekly.com/issues/638) — "Node 26.8 reads and writes ZIP files"
>
> 注記: nodeweekly.com は本セッションのネットワークegressポリシーにより直接アクセスがブロックされているため、RSSフィード (`https://cprss.s3.amazonaws.com/nodeweekly.com.xml`) 経由で号の全文を取得し、各記事のリンク先（nodejs.org、GitHub、各プロジェクト公式サイト等）を個別に調査した。一部の一次情報源（ブログドメイン等）も同様にegressブロックされており、その場合はGitHub上のドキュメント/リリースノート、または複数のWebSearchクエリで一貫して得られたスニペットを情報源として使用し、その旨を明記した。
> "sponsor" 表記のある記事（AgentField.ai、xyOps）は指示により除外した。

---

## 1. Node.js Interactive 2026 リキャップ

**原文リンク:** [A Recap of Node.js Interactive 2026](https://nodejs.org/en/blog/events/nodejs-interactive-2026)（著者: Aviv Keller）

### ① 目的・問題意識

Node.js Interactive カンファレンスが2019年以来7年ぶりに復活し、2026年8月12〜13日にAtlantaのRenderATL / Atlanta Tech Week内で開催された。Node.jsコントリビュータと招待講演者が、AI時代におけるNode.jsの近未来ロードマップ（AI、サプライチェーンセキュリティ、標準化、QUIC/HTTP3対応、リリースサイクルの見直し）を議論する場となった。

### ② 具体的な技術詳細（セッションごと）

- **Robin Bender Ginn (OpenJS Foundation)** — 「Node.js Runs AI. Who Runs Node.js?」。JSエコシステムがAIブームのインフラの大部分を支えているにもかかわらず、メンテナは「見えない存在（missing middle）」として扱われがちだと指摘。

- **Joe Sepi (Cloudflare)** — JSガバナンス（W3C/Ecma/OpenJS）の相関図を解説。**WinterCGが正式な標準化トラックであるEcma TC55（通称WinterTC）に格上げ**され、Node.js/Deno/Cloudflare Workers等サーバーサイドJS実行系の「検証可能な最小共通API」を定義することが目標になった。

- **Kate Holterhoff** — サプライチェーン攻撃の系譜を整理。
  - 2025年9月: Shai-Hulud攻撃（npmアカウント乗っ取り、500超パッケージ削除）
  - 2025年11月: 第二波（796パッケージ、週2000万DL規模）
  - 2026年3月: Axios乗っ取り（OIDCベースの正規リリースフローを迂回し直接侵害）
  攻撃の共通パターンは「人のアイデンティティ侵害 → 正規の公開権限の悪用」。対策として npm trusted publishing、短命クレデンシャル、依存関係のクールダウン期間などが挙げられた。AIの両義性にも言及：悪用面ではソーシャルエンジニアリングのコストが下がる一方、防御面ではAI駆動でOpenSSLの脆弱性12件（うち25年以上潜伏していたバグも）を発見した事例が紹介された。

- **Darcy Clarke (vlt)** — 「Beyond SemVer」。SemVer 2.0.0の仕様上、ビルドメタデータ（`+`以降の部分）が優先度計算で無視される点を逆手に取り、後方互換な拡張ポイントとして使う新仕様 `semver.xyz` を紹介。

- **Matteo Collina (Platformatic)** — Node.jsに既に組み込まれている機能（TypeScript型ストリッピング、`node:test`、`.env`読み込み、Undici駆動`fetch`、watch mode、`node:sqlite`、Permission Model）を整理した上で、**メジャーリリースの年1回化**を発表：
  - Node.js 27.xから、従来の年2回リリースを年1回に移行
  - 2026年10月にアルファ開始 → 2027年4月にCurrent化 → 2027年10月にLTS入り
  - バージョン番号を初回Current化のカレンダー年と揃え、最終的に全リリースがLTSになる設計

- **James Snell (Cloudflare)** — 「QUIC in Node.js」。QUIC（RFC 9000、UDP上の多重化トランスポート）とHTTP/3（RFC 9114、QUIC上へのHTTPマッピング）を明確に区別。現状は `--experimental-quic` フラグ＋QUICサポート付きビルドが必要で、APIモジュール名は `node:quic`。大規模な変更セットの停滞を打開するためAI支援開発を活用したことにも言及。将来的に `fetch` を含むネットワークスタック全体をHTTP/3対応にし、**HTTP/1.1・2・3を横断する統一サーバーAPIをWinterTCに提案する計画**（本文書 §5「Fetch Server API提案」に直結）。

- **Brian Muenzenmeyer & Claudio Wunder** — ドキュメント基盤刷新ツール `doc-kit` を紹介。Markdown解析からWeb/HTML/manページ/JSON/検索インデックス/`llms.txt`までを単一パイプラインで生成する。新しいリーダー体験のプレビューが `beta.docs.nodejs.org` で公開。

- **Ethan Arrowood (Harper)** — 並列実行可能な統合テストツール `@harperfast/integration-testing` を紹介。

- **Andrea Griffiths / Aileen Villanueva Lecuona / Charlie Gerard / Jenna Zeigen** — AI時代の開発者責任、MCP/Skills/仕様駆動開発、INP（Interaction to Next Paint）やTime-to-First-Tokenといった新しい計測指標について議論。

### ③ なぜそれが効果的か

WinterCGのTC55格上げは、Node.js固有の実装に依存しない「サーバーサイドJSの共通言語」を標準化トラックに乗せることで、Node/Deno/Bun/Workers間のポータビリティを保証する制度的な一歩である。またサプライチェーン攻撃の系譜整理は、攻撃が技術的脆弱性ではなく「アイデンティティ層」を狙っている点を明確にし、対策の焦点をtrusted publishingや短命クレデンシャルへ向ける根拠になっている。年1回リリースサイクルへの移行は、AIエージェントによる自動アップグレード作業が増える中で、破壊的変更の頻度を下げてエコシステム全体の追従コストを下げる狙いがある。

---

## 2. Node.js 26.8.0 (Current) Released — ZIPアーカイブAPI & REPL構文ハイライト

**原文リンク:** [Node.js 26.8.0 (Current) blog](https://nodejs.org/en/blog/release/v26.8.0) / 実装PR [nodejs/node#64339](https://github.com/nodejs/node/pull/64339) / REPL PR [nodejs/node#64591](https://github.com/nodejs/node/pull/64591) / [Node.js 26.8.1 out-of-band blog](https://nodejs.org/en/blog/release/v26.8.1)

### ① 目的・問題意識

これまでNode.jsでZIPファイルを扱うには `adm-zip` や `yauzl` 等のサードパーティ依存が必須だった。`node:zlib` にZIPの読み書きAPIをネイティブ追加することで、依存ゼロでZIPアーカイブを扱えるようにする（実装: Philipp Dunkel、2026年8月1日マージ、experimental）。また、REPLの視認性向上のため、既存の `util.inspect` 用カラーパレットを流用した構文ハイライトを追加（著者: Aviv Keller）。

### ② 具体的なコード・技術詳細

新設されたクラス構成:
- **`ZipBuffer`**: 既存の `Buffer` 上のゼロコピー・インメモリビュー。`add()/addSync()`, `delete()`, `toBuffer()/toBufferSync()`, `get()/has()/entries()/keys()/values()`、プロパティ `comment/size/writable`。
- **`ZipEntry`**: アーカイブ内の1メンバー。静的メソッド `create()/createSync()/createStream()/createSymlink()/read()`。インスタンスプロパティ `name/size/compressed/compressedSize/crc32/method/modified/comment/mode/flags/isFile/isDirectory/isSymlink` 等、`content()/contentSync()/contentIterator()`。
- **`ZipFile`**: ディスク上アーカイブへのランダムアクセスビュー。`open()/openSync()`、`add()/addEntry()/delete()/get()/stream()/compact()/close()`。
- 関数: `zlib.createZipArchive(entries[, options])` / `createZipArchiveSync()`、`zlib.zipFiles(files[, options])`、`getMaxZipContentSize()/setMaxZipContentSize(size)`。

公式ドキュメント `doc/api/zlib.md` に掲載されているコード例（すべてverbatim）:

```mjs
// 1. ZipBuffer の読み取り・変更
import { ZipBuffer } from 'node:zlib';
import { readFileSync, writeFileSync } from 'node:fs';
import { Buffer } from 'node:buffer';

const zip = new ZipBuffer(readFileSync('archive.zip'));
for (const [name, entry] of zip) {
  console.log(name, entry.size);
}
await zip.add('hello.txt', Buffer.from('Hello, world!'));
zip.delete('unwanted.txt');
writeFileSync('archive.zip', await zip.toBuffer());
```

```mjs
// 2. ZipFile: 読み取り・ストリーミング・書き込み
import { ZipFile } from 'node:zlib';
import { Buffer } from 'node:buffer';

const zip = await ZipFile.open('archive.zip', { writable: true });
try {
  const entry = await zip.get('member.txt');
  console.log((await entry.content()).toString());
  for await (const chunk of await zip.stream('huge.bin')) {
    // Process each chunk without buffering the whole member.
  }
  await zip.add('new.txt', Buffer.from('hello'));
  await zip.delete('unwanted.txt');
} finally {
  await zip.close();
}
```

```mjs
// 3. ZipEntry から新規アーカイブを作成
import { createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { Buffer } from 'node:buffer';
import { ZipEntry, createZipArchive } from 'node:zlib';

const entries = [
  await ZipEntry.create('hello.txt', Buffer.from('Hello, world!')),
  await ZipEntry.create('data/', Buffer.alloc(0)),
];
await pipeline(
  createZipArchive(entries, 'created by node:zlib'),
  createWriteStream('archive.zip'),
);
```

```mjs
// 4. baseOffset を使った自己展開型アーカイブ（先頭にシェバン等を付与）
import { createWriteStream } from 'node:fs';
import { Buffer } from 'node:buffer';
import { ZipEntry, createZipArchive } from 'node:zlib';

const prefix = Buffer.from('#!/bin/sh\nexit 0\n');
const entries = [await ZipEntry.create('hello.txt', Buffer.from('Hello, world!'))];
const out = createWriteStream('self-extracting.zip');
out.write(prefix);
createZipArchive(entries, { baseOffset: prefix.byteLength }).pipe(out);
```

```mjs
// 5. zipFiles: ディスク上の複数ファイルから直接アーカイブ作成
import { zipFiles } from 'node:zlib';
import { createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';

await pipeline(
  zipFiles([
    ['/data/report.pdf', 'report.pdf'],
    ['/data/notes.txt', 'docs/notes.txt'],
  ]),
  createWriteStream('archive.zip'),
);
```

同機能はPR #65007, #65016, #65278等で堅牢化（悪意ある入力への対策、曖昧なZIP終端の拒否、一時停止イテレータの待機回避）が継続中で、ZIP64にも対応。将来的にはアプリケーションバンドリング（アーカイブから直接ロード）への応用も視野に入っている。

**REPL構文ハイライト**（PR #64591 本文より）:
> "Adds basic syntax highlighting similar to our `util.inspect`. Highlighting is done using `util.inspect.styles`, so the coloring is in-tune with inspect... **Notable Change**: The REPL now performs syntax highlighting when `useColors: true`"

新規依存ライブラリを追加せず、既存の `util.inspect.styles` のカラーパレットを流用してキーワード・文字列・数値等をハイライトする実装。REPLを `useColors: true` で初期化すると自動的に有効化される。

その他26.8.0の変更点: crypto の Cipher/Decipher API で SIV・GCM-SIV モードが有効化、ルート証明書をNSS 3.126に更新、`net.BlockList` のパフォーマンス改善、`perf_hooks` histogram への統計的仮説検定の追加、SQLite `StatementSync.prototype.close()` / `[Symbol.dispose]()`、非スロー版 `MIMEType.parse()` 追加、`TracingChannel` の stable 昇格。

**v26.8.1**（out-of-bandパッチ）: `node --version` がアルファ版扱いのバージョン文字列を誤って報告するバグを修正する緊急リリース。2コミットのみ：`src: revert accidental alpha designation`（PR #65568）と `tools: fix tools/nix/list-requisites.sh`（PR #65560）。

### ③ なぜそれが効果的か

ZIP読み書きのネイティブ対応は、ビルドツールやアプリケーションバンドラ、デプロイパッケージング等で頻出するユースケースに対し、サードパーティ依存とそのメンテナンスコスト・セキュリティリスクを排除する。特に `ZipFile` のストリーミングAPI（`stream()`）は巨大メンバーをメモリに全展開せず処理できる点、`baseOffset` オプションは自己展開型実行可能ファイル（シェバン付きZIP）の作成を可能にする点が実務上効果的。REPL構文ハイライトは既存の `util.inspect` パレットを再利用することで実装コストと視覚的一貫性を両立している。

---

## 3. Node.js 24.20.0 "Krypton" (LTS) バックポート機能

**原文リンク:** [Node.js 24.20.0 (LTS) blog](https://nodejs.org/en/blog/release/v24.20.0)

### ① 目的・問題意識

Node.js 26系（Current）で先行実装された実験的機能のうち、実用性の高いものをLTS（24系）にバックポートし、安定版を使う本番環境からも早期に恩恵を受けられるようにする。

### ② 具体的なコード・技術詳細

#### (a) Package Maps — `--experimental-package-map`（PR #62239、著者: Maël Nison / arcanis）

**解決する問題**: 従来の `node_modules` 解決アルゴリズムの構造的欠陥——phantom dependencies（意図しない推移的依存の解決）、peer dependency 解決失敗（モノレポでの複数バージョン共存）、ホイスティングによる情報の欠落、そして解決のたびにファイルシステムを叩くI/Oオーバーヘッド。PR本文より引用: "Resolution requires I/O - you have to hit the filesystem to resolve packages, and that quickly adds up."

`package-map.json` の形式:
```json
{
  "packages": {
    "my-app": {
      "path": "./src",
      "dependencies": {
        "lodash": "lodash",
        "react": "react"
      }
    },
    "lodash": { "path": "./node_modules/lodash" },
    "react": { "path": "./node_modules/react" }
  }
}
```

解決アルゴリズム: (1) importしているファイルの所属パッケージを特定 → (2) そのパッケージの `dependencies` でspecifierを検索 → (3) 見つかれば対象の `path` に解決 → (4) マップ内の別の場所にあれば `ERR_PACKAGE_MAP_ACCESS_DENIED` → (5) マップに存在しなければ `MODULE_NOT_FOUND`。

```bash
node --experimental-package-map=./package-map.json app.js
# パッケージマネージャは環境変数経由で連携可能
NODE_OPTIONS="--experimental-package-map=$PWD/node_modules/.package-map.json"
```

#### (b) パーミッション監査 — `--permission-audit`（PR #61869）と `permission.drop()`（PR #62672）

`--permission-audit`: アクセスを拒否せず、違反のみ `node:diagnostics_channel` 経由で通知する監査モード（`ERR_ACCESS_DENIED` は投げない）。関連チャンネル: `node:permission-model:fs` / `child` / `worker` / `inspector` / `wasi` / `addon`。

```js
const diagnostics_channel = require('node:diagnostics_channel');

diagnostics_channel.channel('node:permission-model:fs').subscribe((msg) => {
  console.log(`Permission denied: ${msg.permission} on ${msg.resource}`);
});

const fs = require('node:fs');
fs.readFileSync('/etc/passwd');
```

`permission.drop()`: 実行時に権限を不可逆に削除する。参照なしで呼べばスコープ全体、参照を渡せば特定リソースのみ削除。制約として「将来のアクセスチェックにのみ影響し、既に開いているリソースへのアクセスは取り消されない」「明示的に付与された正確なリソースのみdrop可能」。

```js
const fs = require('node:fs');

const config = fs.readFileSync('/etc/myapp/config.json', 'utf8');

process.permission.drop('fs.read', '/etc/myapp');
process.permission.has('fs.read', '/etc/myapp/config.json'); // false
process.permission.drop('child');
```

#### (c) `node:stream/iter`（PR #62066、著者: James M Snell、`--experimental-stream-iter` フラグ）

Web互換で高性能な「新ストリーム」抽象を目指す実験的モジュール。push/pull/transform操作とバックプレッシャー処理（デフォルトはstrictモードで、Web Streamsと異なりバックプレッシャー無視を許さない）を提供する。

```mjs
import { Readable } from 'node:stream';
import { text, from } from 'node:stream/iter';
```

（`--experimental-stream-iter` 有効時、`Readable` は `Symbol.for('Stream.toAsyncStreamable')` プロトコルを実装し、`stream/iter` から効率的に消費可能になる）

ベンチマーク: classic streams比で identity 1MB操作が **2.5倍**、64MB変換が1.1倍、pipeTo 1MBが **2.4倍** 高速。`FileHandle` 統合デモ（読み込み→大文字変換→圧縮）ではNode.js streamsと同等、Web Streamsの2倍速。

#### (d) WebAssembly JSPI（PR #59941、著者: Guy Bedford、2025年9月23日マージ→24.20.0へバックポート）

WebAssembly JS Promise Integration（TC39 Stage 4提案）を有効化。WasmコードとJS非同期処理（Promise）のシームレスな連携を可能にする。既にChromeで出荷済みの機能であり、互換性維持のための対応。

その他: NSS 3.125へのルート証明書更新、npm 11.19.0 / ngtcp2 1.25.0 / nghttp3 1.18.0 / sqlite 3.53.4 / タイムゾーン2026cへの依存更新、test runnerの `context.log()`。

### ③ なぜそれが効果的か

Package Mapsは、パッケージマネージャがビルド時に事前計算した解決マップをNode.jsのモジュール解決に直接使わせることで、実行時のファイルシステムI/Oを削減し、phantom dependencyのような不正な解決経路を構造的に排除する。パーミッション監査モードは、Permission Model導入時の「まず違反を可視化してから段階的に強制する」という移行パスを提供し、本番投入前のリスク評価を容易にする。`node:stream/iter` はNode.js streamsの複雑なAPIサーフェスをWeb標準に寄せつつ、ベンチマークで示された通り既存実装より高速化されている。

---

## 4. fs.glob のC++ネイティブ再実装

**原文リンク:** [nodejs/node#65392 "fs: implement glob natively"](https://github.com/nodejs/node/pull/65392)（著者: Aviv Keller / avivkeller、2026年8月18日オープン、未マージ）

### ① 目的・問題意識

既存の `fs.glob` はJavaScript実装で、グロブパターンを正規表現にコンパイルする際にJS側で大量の文字列アロケーションが発生し、大量ファイルリスト処理時に不要なGCオーバーヘッドを生む。これをC++でlibuvに直接フックするネイティブ実装に置き換え、V8を完全にバイパスしてパターンマッチングを行うことで高速化する。

### ② 具体的な技術詳細

パターン用のAST（抽象構文木）を手動構築してネイティブにマッチングする方式を採用（正規表現へのコンパイルは行わない）。構成ファイル:
- `glob_parser.cc` — パターンコンパイル
- `glob_matcher.cc` — マッチング
- `glob_walker.cc` — libuv経由のディレクトリ走査
- `glob_program.cc` — AST表現
- `glob_unicode.cc` — Unicode処理

既存glob実装と1:1互換（非破壊的変更）。PR本文の注記（verbatim）: "(AI assistance used to help port minimatch to C++ and take improvements from other implementations, e.g. Rust's fast-glob [oxc] and picomatch, etc)." — コミットフッターに `AI-Assisted-By: Claude Fable 5, Claude Opus 5` の記載あり。

ベンチマーク結果の一部（`n=1000`、`***` は統計的有意性大）:
```
fs/bench-glob-options.js options='none' mode='callback' pattern='**/*.js' dir='lib' n=1000     ***    153.86 %
fs/bench-glob-options.js options='none' mode='promise'  pattern='**/*.js' dir='lib' n=1000     ***    150.49 %
fs/bench-glob-options.js options='none' mode='sync'     pattern='**/*.js' dir='lib' n=1000     ***     98.16 %
fs/bench-glob.js mode='callback' pattern='**/*.{js,json}' dir='lib' n=1000                     ***    181.99 %
fs/bench-glob.js mode='callback' pattern='**/*.js'        dir='lib' n=1000                     ***    162.05 %
fs/bench-glob.js mode='callback' pattern='**/*'           dir='lib' n=1000                     ***    167.88 %
fs/bench-glob.js mode='promise'  pattern='**/*.{js,json}' dir='lib' n=1000                     ***    167.71 %
fs/bench-glob.js mode='promise'  pattern='**/*.js'        dir='lib' n=1000                     ***    158.22 %
fs/bench-glob.js mode='sync'     pattern='**/*.{js,json}' dir='lib' n=1000                     ***    112.35 %
```
PR説明文の要約: "This implementation is over 2x faster than the old one."

**議論の詳細**: メンテナ @jasnell の88コアマシンでの追試では非同期158〜528%、同期41〜251%の改善幅を確認。一方、node-glob作者 @isaacs は非同期性能がまだ大きく劣ると指摘（シリアルなディレクトリ走査になっている可能性）。@aduh95 は既存のネイティブglob実装採用の是非を質問し、著者は「成熟した実装はスピードのためにminimatch準拠を犠牲にしがちだが、本実装は互換性を維持したまま高速化した」と回答。@mcollina はminimatchテストの移植不足を指摘し、後に対応された。全既存テストは回帰なくパス。

### ③ なぜそれが効果的か

グロブマッチングは大量ファイルを扱うビルドツール（テストランナー、バンドラ等）で頻繁に呼ばれるホットパスであり、正規表現コンパイルに伴うJS側のアロケーションとGC負荷を排除することは、Node.jsを内部で使うツールチェーン全体のパフォーマンスに波及する。既存API・既存パターン言語との1:1互換を保ちながら2倍以上の高速化を達成している点が、破壊的変更なしの性能改善として評価できる。

---

## 5. Fetch Server API 提案

**原文リンク:** [nodejs/node#65314 "Request for early feedback: Fetch server API"](https://github.com/nodejs/node/issues/65314)（著者: James Snell / jasnell、2026年8月15日オープン）／関連: [jasnell/proposal-fetch-server](https://github.com/jasnell/proposal-fetch-server)／背景issue [nodejs/node#63096](https://github.com/nodejs/node/issues/63096)

### ① 目的・問題意識

Node.jsの `http`/`http2`/`http3`/`quic` の各APIはそれぞれ別の形状を持ち、アプリケーションコードがHTTPバージョンごとの差異を意識する必要がある。これを解消し、**ランタイムに依存しない（Node.js固有ではない）**サーバーAPIをWinterTC/TC55に提案するのが目的。Deno の `Deno.serve` や Bun の `Bun.serve` に相当する高レベルAPIを、標準の `Request`/`Response` 型のみを扱う形で統一する。

Issue本文（verbatim、抜粋）:
> "As part of the new unified http API strategic initiative, I am working on a new WinterTC/TC55 proposal for a new "Fetch Server API". A key goal of this is a server API that is runtime agnostic; and in particular not-Node.js specific. It also covers all versions of http (h1, h2, h3)... This is *NOT YET* an official TC55 work item. It is a draft proposal that I will be presenting to the committee at the next meeting."

### ② 具体的な技術詳細

設計原則（`proposal-fetch-server` READMEより要約）:
- 標準の `Request`/`Response` 型のみを扱う（サブタイプやduck typing不可）
- HTTPメッセージとサーバー処理コンテキスト（`ServerContext`、接続メタデータやライフサイクル管理）を明確に分離
- アプリケーションコードはHTTPバージョンを意識しない（バージョン差異はランタイムの責務）
- ハンドラがサーバーコンテキストを無視しても動作する（オプトイン拡張）
- 準拠ランタイム間でハンドラコードがそのまま移植可能
- extended CONNECTにより将来のプロトコルにもAPI変更なしで対応
- 仕様は2層構造: 必須の「コア層」（ハンドラモデル＋コンテキスト）と、オプションの「インフラ層」（`serve()`、`Server`、ライフサイクル管理）。Cloudflare Workersはコア層のみ実装、Node.jsとDenoは両層を実装予定
- Fetch, Streams, WebTransport、および複数のHTTP関連RFC（優先度制御、datagram、WebSocketアップグレード、プロキシプロトコル）と統合

関連する既存issue #63096（38 reactions）で提示されていた、既存 `http.createServer` と共存する高レベル代替案のAPI形状:
```ts
http.serve((req) => new Response("ok"))
// handler: (Request) => Response | Promise
```

### ③ なぜそれが効果的か

現状、Node.js/Deno/Bun/Cloudflare Workersはそれぞれ独自のサーバーAPI形状を持ち、フレームワーク作者はランタイムごとにアダプタ層を書く必要がある。標準の `Request`/`Response` のみを扱い、HTTPバージョンをランタイム側の責務として隠蔽する設計にすることで、同一のハンドラコードが複数ランタイム間でそのまま移植可能になり、エコシステム全体の断片化を解消できる。2層構造（コア層/インフラ層）の分離は、エッジランタイムのような制約された実行環境でも最小限の実装で準拠できる柔軟性を持たせている。

---

## 6. Node.js公式AI利用ポリシー

**原文リンク:** `doc/contributing/ai-guidelines.md`（nodejs/node、mainブランチ）— OpenJS Foundation AI Coding Assistants Policyに準拠

### ① 目的・問題意識

AIコーディングアシスタントの利用が急増する中、「意思決定は常に人間の判断に基づくべきであり、それが自動化されている場合も同様」という原則を明文化し、AI生成コントリビューションの質を担保しつつ、レビュアーの時間を浪費する低品質PRやbotによる自動PR濫用を防ぐために策定された。

Node.js本文（verbatim）:
> "In the Node.js project, decision making should always be based on human judgement instead of machine automation, regardless of whether the automation is powered by AI. Contributors must take full responsibility for their actions in the Node.js project."

### ② 具体的なガイドライン内容

**AI生成コードの理解・検証義務**（verbatim）:
> "Pull requests with AI-generated code that the contributor has not personally understood, tested, and verified waste collaborator time and will be subject to closure without additional review. Contributors who repeatedly submit such changes, show no understanding of the project or its processes, or are dishonest about the use of automated assistance may be blocked from further contributions."

→ 理解・テスト・検証していないAI生成コードのPRはレビューなしでクローズされうる。繰り返す場合や不誠実な場合はブロックされうる。

**Bot自動PR禁止**（verbatim）:
> "Pull requests must not be opened by automated tooling, unless specifically approved in advance by the project. To request approval, either open an issue in nodejs/admin, or if the automation can be done in the form of a GitHub workflow, submit a pull request to add the workflow..."

→ 自動化ツールによるPRオープンは事前承認が必須（`nodejs/admin` へissueを立てるか、GitHub workflowとしてPRを提出する）。

**"good first issue" の保護**（verbatim）:
> "It's prohibited to use AI to automate fixes to issues marked as 'good first issue'. These issues are meant to help new human contributors, not an AI, learn about the code base and the contribution processes."

→ "good first issue" タスクをAIで自動的に解決することは禁止（新規の人間コントリビューター向けの学習用に確保）。

**その他の実務ガイドライン**:
- 開示時の商標名の扱い: 営利ブランド名（特定AIモデル名等）はコミットメッセージで避け、「a frontier reasoning model」等の匿名表現を推奨（非営利ツールは対象外）。
- AIの分析は「事実」ではなく「仮説」として扱い、実ソースで検証すること。
- 既存テストの削除・変更は人間の検証なしに行わない。
- AI生成コメントは正確性を検証し、コードから自明でない文脈（経緯・動機）を提供するものにする。
- レビュー対応は本人の判断で行い、AIによる自動返信のみで対応できない場合はPRをクローズすべき。
- **AIが全文生成したメッセージをそのまま貼り付けることは禁止**（モデレーションポリシーに基づき削除されうる）。
- 議論でのAI由来の主張は本人が検証し、AI要約ではなく実際のコード・ドキュメント・仕様へのリンクを提示すること。文法・スペルチェックツールの利用は許容される。

### ③ なぜそれが効果的か

このポリシーは「AI利用そのものの禁止」ではなく「検証責任の所在の明確化」に焦点を当てている点が実効性の鍵である。理解・テスト・検証を経ないAI生成PRを機械的にクローズ可能にすることで、レビュアーのボトルネックを構造的に防ぎ、botの自動PR濫用を事前承認制で抑止する。また"good first issue"の保護は、AIエージェントが低難度タスクを先取りしてしまうことで新規コントリビュータのオンボーディング経路が失われる、というOSSコミュニティ特有のリスクに対する具体的な防御策になっている。

---

## 7. Node.js 年次ユーザー調査

**原文リンク:** [SurveyMonkeyフォーム](https://linuxfoundation.surveymonkey.com/r/nodejs-users-2026)（Linux Foundation運営）／背景: [Collaboration Summit 2026 (London) レポート](https://nodejs.org/en/blog/events/collab-summit-2026-london)

### ① 目的・問題意識

Node.jsコミュニティの利用実態・ニーズを継続的に把握し、機能の優先度やリリース方針といった戦略的意思決定に反映するための年次調査。

### ② 具体的な技術詳細

この調査はNode.jsの「Next 10」戦略チーム（コミュニティの今後10年の方向性を議論するグループ）が主導し、Linux FoundationのSurveyMonkey上で実施されている。2026年4月のCollaboration Summit（ロンドン）でJacob SmithとMarco Ippolitoが2025年調査結果（コラボレーターの健全性指標の前年比較、ユーザー調査ハイライト）をレビューし、AIに関する最近の議論を踏まえて2026年版の質問項目（追加・削除・修正候補）をオンサイトで検討・確定させた。ニュースレターが伝える「最新の年次調査がローンチされた」は、このプロセスを経て2026年8月に公開されたアンケートを指す。

### ③ なぜそれが効果的か

リリースサイクルの見直しやAI利用ポリシーなど、直近でNode.jsプロジェクトが下した意思決定の多くはコミュニティの実態調査に基づいている。定点観測的な年次調査により、機能の優先順位付けが一部のコアコントリビュータの主観ではなく、実際のユーザー層のフィードバックに基づいたものになる。

---

## 8. AWS Lambda Node.js 26 ランタイム（プレビュー）

**原文リンク:** AWS Compute Blog「Introducing public preview runtimes on AWS Lambda, starting with Node.js 26 and Python 3.15」／AWS What's New（2026年8月）

### ① 目的・問題意識

AWS Lambdaが「マネージドランタイムのパブリックプレビュー」制度を新設し、その第一弾としてNode.js 26とPython 3.15を対象にした。GA（一般提供）前に顧客・パートナー・上流言語コミュニティが早期に試せるようにし、フィードバックをLambdaチームに提供できるようにする狙い。

### ② 具体的な技術詳細

**利用方法**:
- ZIPデプロイの場合: ランタイムに `nodejs26.x` を選択
- コンテナイメージ（OCI）の場合: プレビュー用ベースイメージ `public.ecr.aws/lambda/nodejs:26-preview`

**提供リージョン**: 全AWS商用リージョン、AWS GovCloud (US)、中国リージョン

**料金・タイムライン**: 追加コストなし（標準Lambda料金で課金）。Node.js 26のアップストリーム安定版リリースは2026年10月予定、Lambdaでの正式GAはそのリリース後2ヶ月以内を目標。

**重要な制約**: プレビューランタイムはフィードバック・検証目的限定であり、本番ワークロードでの使用は非推奨。AWSテクニカルサポート対象外、Lambda SLA（サービスレベルアグリーメント）の対象外。

関連して `aws/aws-lambda-nodejs-runtime-interface-client` リポジトリのissue #198「[FEEDBACK REQUEST] Node.js 26 preview」でも同様のフィードバック募集が行われている。

### ③ なぜそれが効果的か

従来、新しいNode.jsメジャーバージョンはLambdaでのサポート開始までにGA後しばらくのタイムラグがあった。パブリックプレビュー制度により、上流の安定版リリース前からLambda環境での互換性検証が可能になり、Lambda利用者にとってのアップグレード計画の前倒しと、AWS側にとっての早期バグ発見の両方を実現する。

---

## 9. Query Pipelining in node-postgres: 2-3x More Throughput

**原文リンク:** [blog.platformatic.dev の記事](https://blog.platformatic.dev/query-pipelining-in-node-postgres-2-3x-throughput-with-one-line-of-code)（著者: Matteo Collina、egressブロックのため本文は未検証）／実装 [brianc/node-postgres PR #3652](https://github.com/brianc/node-postgres/pull/3652)／公式ドキュメント `docs/pages/features/pipelining.mdx`（verbatim取得済み）

### ① 目的・問題意識

従来のnode-postgresは、デフォルトでは各クエリが完了する（レスポンスを受け取る）まで次のクエリを送信しない。つまり10個のクエリを送ると、10回分のネットワークラウンドトリップ待ち時間が発生する。PostgreSQL自体はワイヤプロトコルレベルでクエリパイプライニングをサポートしていた（PostgreSQL 14でlibpqのパイプラインAPIとして実装）が、node-postgresはこれを一度も活用していなかった。この問題を、`pipeline: true` という設定を1つ追加するだけで解決する。2026年3月にMatteo Collinaが着手したPR #3652が2026年8月8日にマージされ、`pg@8.23.0` および `pg-native@3.9.0` としてリリースされた。

### ② 具体的なコード（公式ドキュメントより逐語的に再現）

**パイプライニングの有効化（基本）**:
```js
import { Client } from 'pg'

const client = new Client({ pipeline: true })
await client.connect()

const [r1, r2, r3] = await Promise.all([
  client.query('SELECT 1 AS num'),
  client.query('SELECT 2 AS num'),
  client.query('SELECT 3 AS num'),
])

console.log(r1.rows[0].num, r2.rows[0].num, r3.rows[0].num) // 1 2 3

await client.end()
```

**プロトコルの図解**（ドキュメント内のASCII図、逐語）:
```
sequential (default)             pipelined
─────────────────────           ─────────────────────
 client ──Parse──▶ server        client ──Parse──▶ server
 client ◀──Ready── server               ──Parse──▶
 client ──Parse──▶ server               ──Parse──▶
 client ◀──Ready── server        client ◀──Ready── server
 client ──Parse──▶ server        client ◀──Ready── server
 client ◀──Ready── server        client ◀──Ready── server
```

**Poolとの併用**:
```js
import { Pool } from 'pg'

const pool = new Pool({ pipeline: true })

const client = await pool.connect()
// client.pipeline is already true

const [users, orders] = await Promise.all([
  client.query('SELECT * FROM users WHERE id = $1', [1]),
  client.query('SELECT * FROM orders WHERE user_id = $1', [1]),
])

client.release()
```
> 注意書き（原文）: `pool.query()` はクライアントを1クエリ分だけチェックアウトしてすぐ解放するため、パイプライニングの効果がない。複数クエリを送るには `pool.connect()` を使う必要がある。

**エラーの独立性（バッチ内の一部失敗が他に影響しない）**:
```js
const results = await Promise.allSettled([
  client.query('SELECT 1 AS num'),
  client.query('SELECT INVALID SYNTAX'),
  client.query('SELECT 3 AS num'),
])

console.log(results[0].status) // 'fulfilled'
console.log(results[1].status) // 'rejected'
console.log(results[2].status) // 'fulfilled'
```
各クエリの後に `Sync` メッセージが送られ、これがPostgreSQLの拡張クエリプロトコルにおけるエラー境界を区切る仕組みだから機能する。

**名前付きプリペアドステートメントの重複排除**:
```js
const queries = Array.from({ length: 100 }, (_, i) => ({
  name: 'get-user',
  text: 'SELECT * FROM users WHERE id = $1',
  values: [i],
}))

const results = await Promise.all(queries.map(q => client.query(q)))
```
同じステートメント名を持つ複数のパイプライン化クエリがある場合、node-postgresは `Parse` を一度だけ送信し、以降はそのプリペアドステートメントを再利用する。

**グレースフルシャットダウン**:
```js
const client = new Client({ pipeline: true })
await client.connect()

const p1 = client.query('SELECT 1')
const p2 = client.query('SELECT 2')
const endPromise = client.end()

// Both queries will resolve normally
const [r1, r2] = await Promise.all([p1, p2])
await endPromise
```
`client.end()` を呼んでも、進行中のパイプライン化クエリが完了するまで接続クローズを待つ。

### ③ なぜそれが効果的か

- **ラウンドトリップの削減**: 通常は「クエリ送信→サーバーの応答を待つ→次のクエリ送信」という直列処理のため、n個のクエリでn回分のネットワーク往復遅延（RTT）を支払う。パイプライニングでは全クエリを一気に送信し、応答は送信順に読み戻すため、待機時間の大部分を排除できる。
- **並列処理ではなく並行I/O**: 重要な点として、PostgreSQLサーバー側は依然としてクエリを逐次実行する（実行の並列化ではない）。パイプライニングが改善するのはあくまで通信の両側が待たずに済むようにする「ネットワークI/Oの並行性」であり、クエリ実行自体のスループットではない。
- **レイテンシが高いほど効果大**: クライアント-サーバー間のネットワーク遅延が大きいほど、また送信するリクエスト数が多いほど、パイプライニングによる性能向上は大きくなる。
- **Sync境界によるエラー分離**: 各クエリ後に `Sync` メッセージを送ることで、拡張クエリプロトコル上でクエリ単位のエラー境界を維持しつつ一括送信が可能になっている。
- **プリペアドステートメントの重複排除**: 同名の `Parse` をキャッシュ・再利用することで、同一クエリを繰り返し送る場合の準備コストも削減される。

**ベンチマーク数値**: 公式PR #3652記載の定性的説明では「ローカル接続でのシンプルなクエリのバッチにおいて、通常2〜3倍のスループット向上。高レイテンシ環境ではさらに大きな向上」。WebSearch経由で複数回一貫して確認された参考値（本文未検証）:

| クエリ種別 | パイプライニングなし | パイプライニングあり | 倍率 |
|---|---|---|---|
| シンプルな `SELECT 1` | 10,746 qps | 25,294 qps | 2.35倍 |
| 名前付きプリペアドクエリ | 9,983 qps | 22,160 qps | 2.22倍 |
| ネイティブ(pg-native)パラメータ化クエリ | 8,670 qps | 20,792 qps | 2.40倍 |

---

## 10. ESM vs CommonJS: What `import` and `require` Actually Do

**原文リンク:** [Your Modules Are Lying to You](https://blog.gaborkoos.com/posts/2026-08-14-Your-Modules-Are-Lying-to-You/)（著者: Gabor Koos、egressブロックのためWebSearchスニペットの統合による再構成。原文一字一句の引用ではない点に留意）

### ① 目的・問題意識

「`import` と `require` は同じことをする別の書き方だ」という一般的な誤解を扱う。実際には両者は、モジュールを取り込むという一見単純な操作の裏で、以下の複数フェーズで異なる決定をしている：
- モジュールの解決（resolving a module）
- モジュールインスタンスの生成・取得（creating or retrieving its module instance）
- コードの評価（evaluating its code）
- 他モジュールへの値の公開（exposing something to another module）

この分解の違いが、副作用がいつ実行されるか、循環依存で何が観測できるか、2つの消費者が状態を共有するか、あるパッケージが `import` では動くのに `require` では壊れる理由を左右する。

### ② 具体的な技術詳細（検索スニペットからの再構成）

- **ライブバインディング vs コピーされた値**: ESMの名前付きimportは、エクスポート元モジュールが所有する束縛（binding）への参照であり、元モジュール側で値が再代入されると、import側でもその変化が見える。一方CommonJSの `require()` は `module.exports`（通常のオブジェクト）を返すだけなので、それを分割代入するとその時点の値がローカル変数にコピーされ、以後は追従しない。

- **ゲッター経由の公開**: `let count` のようなプライベートな状態を、`Object.defineProperty` によるgetterで公開する例。`counter.count` のようにプロパティに毎回アクセスすればgetterが都度呼ばれ最新値が取れるが、分割代入で一度だけ取り出すとその時点のスナップショットになってしまう。

- **循環依存**: `a.mjs` が `b.mjs` に依存し、`b.mjs` が先に評価される場合。ESMのリンカーは、どちらのモジュール本体も実行される前に、インポートされる束縛をあらかじめ作成する。そのため `b.mjs` から `a` という束縛を読もうとした時点で、束縛自体は存在するが初期化はまだ済んでいない（TDZ = temporal dead zone）状態になり、評価が停止する。これに対しCommonJSの循環参照は、部分的に構築された（初期化未完了の）`exports` オブジェクトを返してしまう。

- **デュアルパッケージハザード**: `package.json` の `exports` フィールドで `import` 条件と `require` 条件に別々のエントリポイントを与えると、バンドラー（webpack等）は重複排除や単一条件の選択によってこの問題を隠してしまうことがあるが、Node.jsネイティブの実行では2つの別々のモジュールインスタンス（したがって別々の状態・別々のクラスID）が生成されてしまう。

- **記事内で提示されている設計指針**:
  1. ステートフルな実装はCommonJS側に置き、ESM側のエントリポイントはそれを再エクスポートするだけにする
  2. 状態を両ラッパーの外側に持たせ、フォーマット別のエントリポイントはステートレスにして状態を注入する
  3. どちらか一方のフォーマットのみを公開し、もう一方はNodeのネイティブ相互運用（`require(esm)` など）に任せる
  4. パッケージのテストは、tarballをビルドして小さな消費側プロジェクトに実際にインストールする形で、公開名経由で選択されるアーティファクトに対して行うべき
  5. `import` と `require` それぞれで選択される別ファイルは、意図的に同じ実装を共有させない限り「独立したモジュール」として設計すべき
  6. 両方のローダーからパッケージに到達しうる場合は、クラスの同一性や設定の共有状態をテストすべき

- **デバッグ指針**: 値がESMのnamespaceオブジェクトなのか、CommonJSのexportsオブジェクトなのか、あるいはコンパイラ（Babel/TypeScript等）が生成したラッパーなのかをまず切り分け、「なぜ `.default` が存在するのか」を理解してから `.default` にアクセスすべき。

### ③ なぜそれが効果的か

ESMの「ライブバインディング」とCommonJSの「値のコピー」という違いを理解していれば、なぜ分割代入した値が古くなる（stale destructured state）のかが分かる。モジュール評価の順序（linkフェーズとevaluateフェーズの分離）を理解すれば、循環依存時に「未初期化の束縛」と「部分的に構築されたexportsオブジェクト」のどちらに遭遇するかを予測でき、TDZエラーの原因を特定しやすくなる。`package.json` の `exports` 条件分岐によって `import` 用と `require` 用で別ファイルが選ばれる仕組みを理解すれば、デュアルパッケージハザード（同一パッケージの2インスタンス化によるシングルトン崩壊やinstanceof失敗）を事前に設計で回避できる。

---

## 11. pnpm 12: Rewritten in Rust, Same Commands and Lockfile

**原文リンク:** [pnpm/pnpm Release v12.0.0](https://github.com/pnpm/pnpm/releases/tag/v12.0.0)（pnpm.io自体は直接アクセス不可のためGitHub Releasesとリリースノートを情報源とする）

### ① 目的・問題意識

pnpm 12はコアをRustで書き直した安定版。目的は「移行」ではなく、v11のコマンド・フラグ・設定・ロックフォーマットをそのまま維持しつつ、Node.jsランチャーを排したネイティブバイナリで起動コストと解決速度を改善すること。ドキュメントもv11/v12共通。

### ② 具体的なコード・変更点（GitHubリリースノートより抜粋）

**Gitホスト依存関係の正規化**（破壊的変更）:
```
Git dependencies on known hosts (GitHub, GitLab, Bitbucket) are now treated
as identities rather than transport choices. All repository representations
resolve through canonical HTTPS URLs.
```
プライベートリポジトリをSSH経由で使う場合は以下のgit設定が必要:
```
git config --global url."git@github.com:".insteadOf https://github.com/
```

**pnpm-workspace.yamlの厳格化**（破壊的変更）: 未知の設定キーがあると、プロジェクトのpin済みpnpmバージョンと一致する場合に `ERR_PNPM_UNRECOGNIZED_WORKSPACE_SETTINGS` エラーを出し、似た有効な設定名を提案する。

**循環依存解決の正規化**:
```
Dependency cycles are now broken canonically during peer resolution: the
members of each cycle are ordered by package id.
```
これによりインストール順序に関わらずロックファイルが決定的になる。cycle-heavyなワークスペースで解決速度2〜3倍、メモリ使用量約25%減。

**Linuxでのハードリンク優先**: `packageImportMethod: auto` がLinuxでclone前にhardlinkを試すようになった。btrfs上でmaterialization時間がほぼ半減。フォールバックはclone。

**engineStrictの挙動変更**: optional依存経由でも非互換パッケージがregular依存に到達するとインストール失敗。

**グローバルシム（重要な新機能 — Node.jsがプロジェクトpin追従）**:
```
globalShims: { node: true, deno: true, bun: true }  // デフォルト
```
グローバルにインストールした `node`/`deno`/`bun` が、プロジェクトがpinしているバージョンに従うようになった（プロジェクトごとに初回「Do you trust this project?」の確認あり）。

**パッケージマネージャのインストール**:
```
pnpm dlx (pnx) でバージョン指定実行:
pnx yarn@4 install
pnx npm@11 ci

プロジェクト対応コマンドリンクの作成:
pnpm shim add yarn
```

**その他マイナー変更**: `pnpm init` が最新pnpmバージョンをレジストリタグからpinするようになった。実験的な `remoteSideEffectsCache`（署名付きアーティファクトでビルド出力をマシン間で再利用、pnpr経由）。`audit.ignorePrune` 設定。`pnpm stage approve` でパッケージ承認をバッチ処理。`<version>+rN` 構文でレジストリのリビジョン選択。sudo操作は `ERR_PNPM_SUDO_NOT_SUPPORTED` で失敗するように。

**削除された機能**（破壊的変更）: `--resolution-only` フラグは廃止。代わりに `pnpm peers check` を使う（ロックファイルを直接読み、再解決しない）。

**アップグレード方法**（npmの`latest`タグはまだpnpm 11を指しているため）:
```
pnpm self-update next-12
```
v11.10.0以降であれば、この1コマンドでRust版v12にネイティブバイナリを直接リンクできる。v12は `@pnpm/exe.<platform>-<arch>` というネイティブバイナリパッケージとして配布され、`pnpm`/`@pnpm/exe` 両方の名前で `next-12` distタグ配下に公開されている。Node.jsランチャーが無いため起動時のNode.js初期化コストがゼロになる。

### ③ なぜそれが効果的か

コマンド/フラグ/ロックフォーマットが不変なので、既存のCI/ワークフローを壊さずに高速化の恩恵だけ得られる。循環依存の正規順序化により、インストール順序に依存しない再現性のあるロックファイルが生成される。Node.jsランチャーを排除したことで、大規模モノレポでの起動オーバーヘッドが実質ゼロになる。`next-12` という別distタグでの提供により、`latest` ユーザーを巻き込まずに希望者だけが早期アップグレードできる設計になっている。

---

## 12. Better Auth 1.7 Ships a Standards-Based OAuth Server

**原文リンク:** [better-auth/better-auth Release v1.7.0](https://github.com/better-auth/better-auth/releases/tag/v1.7.0)（better-auth.comは直接アクセス不可のためGitHubリポジトリのドキュメント/READMEを情報源とする）

### ① 目的・問題意識

Better AuthをフルスペックのOAuth 2.1認可サーバーに変える。DPoP（トークン窃取対策）、デバイスコードフロー（CLI/スマートTV等の入力制約デバイス向け）、リソースごとのトークンルール、MCP（Model Context Protocol）専用の認証パッケージ化など、「自前でOAuthサーバーを実装・運用する」という重い運用課題に対する標準準拠の解決策を提供する。

### ② 具体的なコード

**OAuthサーバーの基本セットアップ**:
```ts
import { betterAuth } from "better-auth";
import { jwt } from "better-auth/plugins";
import { oauthProvider } from "@better-auth/oauth-provider";

const auth = betterAuth({
  disabledPaths: ["/token"],
  plugins: [
    jwt(),
    oauthProvider({
      loginPage: "/sign-in",
      consentPage: "/consent",
    })
  ],
});
```
セットアップ後: `npx auth migrate` または `npx auth generate`

**DPoP（RFC 9449）設定**:
```ts
oauthProvider({
  dpop: {
    proofMaxAgeSeconds: 300,
    signingAlgorithms: ["ES256", "EdDSA"],
  },
  resources: [
    {
      identifier: "https://api.example.com",
      dpopBoundAccessTokensRequired: true,
    },
  ],
})
```
DPoPが有効な場合、トークンレスポンスに `token_type: "DPoP"` が含まれ、以降のリクエストには署名済みproof JWTが必須になる。リソースリクエストは `Authorization: DPoP <access_token>` ヘッダーと、アクセストークンハッシュ(ath)を含むDPoP proofを使う。

**デバイスコードフロー（RFC 8628）**:
```ts
import { oauthDeviceAuthorization } from "@better-auth/device-authorization";

const auth = betterAuth({
  plugins: [
    jwt(),
    oauthProvider({ /* config */ }),
    oauthDeviceAuthorization()
  ],
});
```
`urn:ietf:params:oauth:grant-type:device_code` グラントを登録し、クライアントは `/device/code` でコードをリクエストし `/oauth2/token` をポーリングする。

**リソースごとのトークンルール**:
```ts
oauthProvider({
  resources: [
    "https://api.example.com",
    {
      identifier: "https://api.example.com/mcp",
      allowedScopes: ["mcp:read", "mcp:write"],
      accessTokenTtl: 300,
    },
  ]
})
```

**APIでのトークン検証**:
```ts
import {
  requestToResourceInput,
  verifyAccessTokenRequest,
} from "better-auth/oauth2";

export const GET = async (req: Request) => {
  const payload = await verifyAccessTokenRequest(
    requestToResourceInput(req), 
    {
      verifyOptions: {
        issuer: "https://auth.example.com",
        audience: "https://api.example.com",
      },
      requiredScopes: ["read:post"],
    }
  );
  // 認証済みリクエストの処理を続行
}
```

**MCP認証（独立パッケージ `@better-auth/mcp`）**:
```ts
import { betterAuth } from "better-auth";
import { jwt } from "better-auth/plugins";
import { cimd } from "@better-auth/cimd";
import { mcp } from "@better-auth/mcp";
import { fetchClientMetadataResource } from "@better-auth/cimd/node";

export const auth = betterAuth({
    plugins: [
        jwt(),
        mcp({
            loginPage: "/sign-in",
            consentPage: "/consent",
            resource: "https://api.example.com/mcp"
        }),
        cimd({
            fetchClientMetadataResource,
            metadataProfile: "mcp-2026-07-28",
        }),
    ]
});
```

保護されたMCPルート:
```ts
import { requireMcpAuth } from "@better-auth/mcp";
import { createMcpHandler, McpServer } from "@modelcontextprotocol/server";

const POST = requireMcpAuth(
    auth,
    (request) => mcpServerHandler.fetch(request),
    {
        resource: "https://api.example.com/mcp",
    },
);

export { POST };
```

スコープ制限:
```ts
requireMcpAuth(auth, handler, {
    resource: "https://api.example.com/mcp",
    requiredScopes: ["mcp:tools"],
})
```
（スコープ不足時は `insufficient_scope` チャレンジ付き403でstep-up認可を促す）

リモート配置向け:
```ts
import { createMcpProtectedRequestHandler } from "@better-auth/mcp";

const handler = createMcpProtectedRequestHandler(
    {
        issuer: "https://auth.example.com",
        audience: "https://api.example.com/mcp",
        jwksUrl: "https://auth.example.com/api/auth/jwks",
    },
    async (request, accessTokenClaims) => {
        // 認証済みMCPリクエストの処理
    }
)
```

**破壊的変更（アップグレードガイドより）**:
- `npx auth upgrade` で自動アップグレード、事前に `npx auth migrate plan`
- アカウント識別戦略の変更: `account: { identityStrategy: "provider-id" }`（populated済みDBの推奨戦略）
- `signIn.oauth2()` → `signIn.social()`、コールバックURLが `/api/auth/oauth2/callback/:id` → `/api/auth/callback/:id` に変更
- PKCEがデフォルトで有効化
- `verifyAccessToken` → `verifyBearerToken` にリネーム
- MCPは `better-auth` 本体から `@better-auth/mcp` + `@better-auth/oauth-provider` に分離、エンドポイントが `/mcp/*` → `/oauth2/*` へ移動、ヘルパー `withMcpAuth` → `requireMcpAuth` にリネーム
- `oauthDeviceAuthorization()` は明示的にopt-inするプラグインに（自動有効化されない）
- `deviceCode`/`userCode` にユニークインデックスが必須

### ③ なぜそれが効果的か

DPoP（RFC 9449）・デバイスコード（RFC 8628）・OIDC Core仕様（at_hashクレーム）に厳密準拠しており、既存OAuthクライアント/エコシステムとの相互運用性が高い。DPoPによりアクセストークン単体の窃取では悪用不能になる（クライアント鍵による束縛）。LLMエージェント時代の「MCPサーバーの認可」という新しいユースケースを、OAuthプロバイダの上に薄いレイヤーとして構築しており、リソースごとに個別のトークンルール・スコープ・TTLを設定できる。

---

## 13. GTKX: Build Linux Desktop Apps with Node, GTK and React

**原文リンク:** [gtkx-org/gtkx](https://github.com/gtkx-org/gtkx)（gtkx.devは直接アクセス不可のためGitHub READMEを情報源とする）

### ① 目的・問題意識

GTK4は成熟しているが、GtkBuilder XMLで作った静的インターフェースには「アプリケーション状態が変わっても再レンダリングされず、開発中にホットリロードもされない」という欠落がある。GTKXはGNOMEスタックの上に宣言的なReactレイヤーを載せ、Webの馴染み深いツールとパターンをネイティブLinuxアプリ開発に持ち込む。

### ② 具体的なコード（カウンターアプリのサンプル）

```tsx
import * as Gtk from "@gtkx/gi/gtk";
import { GtkApplicationWindow, GtkBox, GtkButton, GtkLabel } 
  from "@gtkx/jsx/gtk";
import { createRoot, quit } from "@gtkx/react";
import { useState } from "react";

const Counter = () => {
  const [count, setCount] = useState(0);

  return (
    <GtkApplicationWindow
      title="Hello GTKX"
      defaultWidth={400}
      defaultHeight={300}
      onCloseRequest={quit}
    >
      <GtkBox orientation={Gtk.Orientation.VERTICAL} spacing={20}>
        <GtkLabel cssClasses={["title-1"]}>Welcome to GTKX!</GtkLabel>
        <GtkLabel>{`Count: ${count}`}</GtkLabel>
        <GtkButton
          label="Increment"
          onClicked={() => setCount((c) => c + 1)}
          cssClasses={["suggested-action"]}
        />
      </GtkBox>
    </GtkApplicationWindow>
  );
};
```

**インストール/スタート**:
```bash
npm create gtkx
cd my-app
npm run dev
```

**JSXをネイティブGTKウィジェットに橋渡しする仕組み**は多層アーキテクチャ:
1. **Reactリコンサイラ**: 「every GObjectをJSX要素として公開するReactリコンサイラ」がコアの抽象化を担い、Reactコンポーネントツリーをネイティブウィジェット階層に変換する
2. **コード生成**: GObject-Introspectionのメタデータから、TypeScriptバインディングとネイティブFFI呼び出しを生成し、型安全性を確保して宣言とランタイム挙動の乖離を防ぐ
3. **ランタイムブリッジ**: 実行時にはネイティブRustコアが `libffi` を介してGTK4/Adwaitaライブラリと直接通信し、`libgirepository` を完全にバイパスする（webviewは使わない）

**Fast Refresh対応**: スキャフォールディングツールに付属の開発サーバーが「実行中のUIをその場でパッチするFast Refresh」を提供し、フルアプリ再起動なしの反復開発を可能にする。

**要件**: Linux + GTK4（4.20+）、Adwaita（1.8+）、GLib開発ライブラリ、Node.js 24以降。x64/arm64 glibc Linux向けにプリビルドバイナリを提供、それ以外のアーキテクチャはRustツールチェーンでのビルドが必要。

### ③ なぜそれが効果的か

webviewを使わず、JSXが直接実GTK4ウィジェットにマップされるため、OSネイティブの見た目・パフォーマンス・アクセシビリティを維持できる。React/JSXという既存のメンタルモデルとツールチェーンでLinuxデスクトップアプリが書けるため、Web開発者のオンボーディングコストを削減する。GObject-Introspectionからの自動コード生成により、実行時エラーではなくコンパイル時に不整合を検出できる。

---

## 14. Stryker Mutator 10.0: Test Your Tests with Mutation Testing

**原文リンク:** [stryker-mutator/stryker-js Releases](https://github.com/stryker-mutator/stryker-js/releases)（stryker-mutator.ioは直接アクセス不可のためGitHub Releasesを情報源とする）

### ① 目的・問題意識

StrykerJSはコードに意図的な変異（ミューテーション、例: `>` を `>=` に変える、条件式を反転させる等）を加え、既存のテストスイートがその変異を検知（kill）できるかを見る。検知できなければ、そのテストは「甘い」（=本来のバグも見逃す可能性がある）ことを意味する。カバレッジ率（行を実行したか）だけでは分からない、テストの「実効性」を測るためのツール。

### ② 具体的な技術詳細

**v10.0.0の新機能・変更点**:
- `empty-expression-mutator`（新しいミューテーション種別）
- mutant filtering（ミュータント絞り込み）
- 予期しない終了時に部分的なインクリメンタルレポートを保存する機能
- Mocha v12サポート
- TypeScript@7の実験的サポート
- Babelがメジャーバージョン8に更新
- Svelteテンプレート式パース改善、Mochaランナー強化、Vitest警告の抑制など各種バグ修正

**破壊的変更**:
```
Node.js 20 is no longer supported, please use Node.js 22 or higher.
```

**導入方法**:
```bash
npx stryker init
npx stryker run
```
`stryker init` で対話的にテストランナー（Jest/Mocha/Vitest等）とレポーターを選択しstryker.config.jsonを生成、`stryker run` でミューテーションテストを実行しHTMLレポート（kill/survived/no coverage等の内訳）を出力する。

### ③ なぜそれが効果的か

「カバレッジ100%だがバグを検知できないテスト」を機械的にあぶり出せる。survived mutant（検知されなかった変異）は具体的にどの分岐・条件のテストが不足しているかを示すため、テスト強化の優先順位付けに使える。mutation scoreを閾値としてCIに組み込むことで、テストスイートの劣化を防止できる。

---

## 15. SVGO 4.1: Node-Powered SVG Optimizer

**原文リンク:** [svg/svgo Releases](https://github.com/svg/svgo/releases)（svgo.devは直接アクセス不可のためGitHub Releasesを情報源とする）

### ① 目的・問題意識

SVGOはSVGファイルからメタデータ・冗長データ・コメント等を除去し、視覚的な差異なしにファイルサイズを縮小するCLIツール/Node.jsライブラリ。v4.1はSAXパーサーのアップグレードによる厳格なXML検証と、スクリプト実行を悪用したセキュリティホールの是正が主眼。

### ② 具体的な技術詳細

**厳格なXML検証**: `sax` を1.5.0 → 1.6.1にアップグレード。新パーサーは数値文字参照をXML許可範囲に対して検証するようになった。以下のような不正な参照（テキスト・属性の両方）を拒否:
- 禁止された制御文字（例: `&#1;`）
- UTF-16サロゲートコードポイント（例: `&#xD800;`）
- 無効なXMLコードポイント（例: `&#xFFFF;`）

**破壊的変更**: 以前は受理されていた不正な形式のSVGが、パーサーエラーを出すようになる可能性がある（有効なXMLドキュメントには影響なし）。パーサー失敗は `SvgoParserError` として "Invalid character entity" という理由で表面化する。

**セキュリティ改善（`removeScripts` プラグインの強化）**:
- HTML/XHTML/SVGドキュメントを含む実行可能な `data:` URLをフィルタ（PNG画像のような無害なデータは保持）
- レガシーな `vbscript:` URLをフィルタ
- SVG `<foreignObject>` 要素内のコンテンツをサニタイズ（HTMLイベントハンドラ属性と実行可能URLを除去）
- 名前空間プレフィックス付きSVG `<a>` 要素の処理、URLスキームチェック前にASCIIタブ/改行を除去

これらは以下のセキュリティアドバイザリに対応: **GHSA-4vpr-x523-8j87**、**GHSA-w27v-7q3p-w38r**

その他: `css-select` をv6に、`css-what` をv7にアップグレード。

**CLI使用例**（一般的なSVGO利用法）:
```bash
# 単一ファイル
svgo input.svg -o output.svg

# ディレクトリ一括
svgo -f ./icons -o ./icons-optimized

# 設定ファイル(svgo.config.js)を使う場合
svgo --config=svgo.config.js input.svg
```

### ③ なぜそれが効果的か

`removeScripts` の強化は、悪意あるSVGアップロード（SVG内に埋め込まれたJavaScript実行ベクタ）を除去する「サニタイザーとしてのSVGO」という用途に直結し、data: URL経由のスクリプト実行やforeignObject経由のイベントハンドラ注入といった攻撃面を塞ぐ。XML検証の厳格化により、不正な文字参照を含む壊れたSVGを「静かに」処理してしまうことがなくなり、パイプラインの早期段階で問題を検出できる。

---

## 16. TermDOM: CSS Grid対応

**原文リンク:** [bikeshaving/termdom](https://github.com/bikeshaving/termdom)（termdom.orgは直接アクセス不可のためGitHub READMEを情報源とする）

### ① 目的・問題意識

TermDOMは実際のDOMノードを端末出力に描画し、ノードが変異するたびに画面を再描画するライブラリで、素のJavaScriptや任意のフロントエンドフレームワークでTUI/対話型CLIを書けるようにする。ブラウザのレンダリングパイプラインを、ピクセルではなく文字セルのグリッドに対して実装しているのが特徴。CSSの長さ単位はセルを基準とする（`1px` と `1ch` はどちらも1セルを意味する）。従来はflexboxのみサポートしており、ダッシュボード状の複雑な2次元レイアウトの表現に限界があった。

### ② 具体的な技術詳細

**動作原理**: 毎フレーム、変異した要素のスタイルとレイアウトを再計算 → 結果をセルバッファに描画 → 前フレームと差分比較 → 差分をANSIエスケープシーケンスとしてstdoutに書き込む。入力側は逆方向で、stdinのエスケープシーケンスをキーボード・マウス・ペーストイベントにデコードしDOMノードにディスパッチする。

サポート機能: カスケーディングCSSのスタイルシート、CSSボックスモデル、flexbox、テーブルレイアウト、CJK/絵文字のテキストレンダリング、`querySelector()` 等のDOMユーティリティ、キーボード/マウスイベント、フォームコントロール、スクロール、Web Components、テキスト選択、フルスクリーン。Node/Bun/Denoで動作しネイティブ依存なし。

**CSS Grid対応**（v0.1.5、2026年8月22日公開）。リリースノート原文抜粋:
```
CSS Grid: track sizing, placement and alignment in the layout engine;
grid shorthand parsing and serialization; used track sizes resolved
through the CSSOM.
```
（トラックサイジング、配置・整列のレイアウトエンジン組み込み、grid shorthandのパース/シリアライズ、CSSOM経由で解決される使用トラックサイズ）

※注記: READMEおよびリリースノート本文には、CSS Gridを使った具体的なコード例は掲載されていなかった（READMEにあるのはflexboxベースのプログレスバー例のみ）。examplesディレクトリ（markdown viewer、chat client、TodoMVC、file picker、weather app、popover menus、solitaire game等）には実装例があると思われるが、README上にはファイル参照のみでコード本文は含まれていない。

### ③ なぜそれが効果的か

flexbox止まりだった従来のTUIレイアウトライブラリに対し、CSS Grid（2次元グリッドレイアウト）を追加することで、ダッシュボード状の複雑なTUI（例: ステータスパネル+ログ+サイドバーの3カラム構成など）をブラウザと同じCSSの語彙で組めるようになる。ブラウザの実装（CSSOMでtrack sizeを解決する等）に忠実に追従しているため、Web開発者がそのままの知識を持ち込める。

---

## 17. その他の小さいリリース（簡易まとめ）

- **Electron 44**: 32-bitビルド（Windows ia32、Linux armv7l）廃止、64-bit（x64/arm64）のみに。macOS 13（Ventura）以降が実行要件に。clipboardモジュールがW3C Clipboard API準拠に刷新され非同期化・レンダラープロセスから直接アクセス不可に（`ClipboardItem` クラス導入）。メインプロセス向けWHATWG互換WebSocketクライアント `net.WebSocket` を追加。LinuxでのウィンドウOpacity対応。
- **lz4-napi 2.10.0**: Rustの `napi-rs` と `lz4_flex`（純Rust実装のLZ4）によるN-APIバインディングで、Node.jsから高速なLZ4圧縮/伸長を利用可能にするライブラリ。
- **grammY 1.46.0**: Telegram Bot API 10.3対応のTelegramボットフレームワーク。
- **Axios 1.20.0**: `form-data` の依存下限を `^4.0.6` に引き上げ、CRLFインジェクション脆弱性GHSA-hmw2-7cc7-3qxxの影響を受けるバージョンが新規インストールで解決されないようにした。NO_PROXYマッチング、インターセプターエラー、進捗レポート、シリアライズのエッジケースも修正。
- **node-mongodb-native 7.6**: MongoDB Server 4.2のサポートを終了（最小サポートを4.4に引き上げ）。
- **YouTube.js 18.0**（youtubei.js）: YouTubeの非公開API「InnerTube」向けJSクライアントのメジャーアップデート。
- **node-x11 4.1**: X11プロトコルクライアントライブラリの更新（一次情報は限定的）。

---

## 除外した記事（sponsor表記）

- AgentField.ai「Open Source Just Out-Reviewed the Commercial Tools」（PR-AF、AIコードレビューツール）
- xyOps「Outgrowing Cron? Meet xyOps for Self-Hosted Ops Automation」
