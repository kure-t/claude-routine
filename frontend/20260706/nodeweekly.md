# Node Weekly Issue 629 まとめ ("Extending Node instead of replacing it")

> 出典: [Node Weekly Issue 629](https://nodeweekly.com/issues/629)

---

## 1. Nub — Node.js を置き換えるのではなく拡張する「オールインワン」ツールキット

**原文リンク:** [nubjs.com](https://nubjs.com/) / [Introducing Nub](https://nubjs.com/blog/introducing-nub) / [GitHub - nubjs/nub](https://github.com/nubjs/nub)

この号のタイトル "Extending Node instead of replacing it" の由来となったリード記事。作者は Zod の作者であり元 Bun 社員でもある Colin McDonnell。2026年6月15日リリース。

### ① 目的・問題意識

Bun や Deno は Node.js を「ゼロから作り直したランタイム」で置き換えるアプローチを取るが、これは既存資産（ネイティブアドオン、Node 専用の挙動、社内ツール群など）との互換性リスクを伴う。一方で `npm run`、`npx`、`nvm` といった周辺ツールチェーンは長年 CLI 起動が遅い・体験が分断されている、という問題を抱えていた。Nub は **ランタイム自体は素の `node` のまま**、その手前にある「TypeScript 実行」「パッケージ実行」「パッケージ管理」「Node バージョン管理」だけを単一の Rust 製バイナリで高速に肩代わりする、という設計思想を取る。

この設計を可能にしたのが、Node 本体に最近追加された拡張ポイントである `module.registerHooks()`(同期的なリゾルバフック API) と `--import` フラグで、Hacker News のコメントでも「Nub のような外部ツールを可能にする大きなアンブロッカーだった」と評されている。

### ② 具体的なコードと技術詳細

#### インストール

```bash
curl -fsSL https://nubjs.com/install.sh | bash
irm https://nubjs.com/install.ps1 | iex     # Windows
brew install nubjs/tap/nub
nix run github:nubjs/nub
mise use -g nub
npm install -g --ignore-scripts=false @nubjs/nub
```

```yaml
# GitHub Actions
- uses: nubjs/setup-nub@v0
```

#### ファイル実行 / スクリプト実行 (TypeScript をそのまま実行)

```bash
nub index.ts
nub --watch app.ts

nub run build
nub run -r --filter "@org/*" test
```

#### パッケージ実行 (npx 相当)

```bash
nubx eslint . --fix
nubx -y cowsay@1.5.0 "hi"
nub dlx
```

#### パッケージマネージャ

```bash
nub install
nub ci
nub add -E -D --save-catalog react
nub remove lodash
nub update
nub dedupe
```

#### Node バージョンマネージャ

```bash
nub node which
nub node install [version]
nub node ls
nub node uninstall <version>
nub node pin <version>

nub upgrade   # nub 自体の自己更新
```

#### 実行例（プロジェクトの Node バージョンを自動解決）

```
$ echo 26 > .node-version
$ nub hello.ts
Using Node.js 26.3.0 (resolved from .node-version)
Installed in 9.8s
Hello world!
```

Node バージョンの解決優先順位は `NODE_EXECUTABLE` 環境変数 → `package.json#devEngines` → `.node-version` → `.nvmrc` → `package.json#engines` の順。TypeScript/JSX/デコレータの変換には `oxc`（Rust 製、Node アドオンとしてコンパイル）を使い、変換後のコードをプロジェクトに固定された素の Node バイナリ上で実行する。

#### セキュリティ機構

- postinstall スクリプトはデフォルトでブロック
- 依存解決時に osv.dev への脆弱性照会を実施
- provenance のダウングレードを拒否
- デフォルトで 24 時間の `minimumReleaseAge`（公開直後のパッケージを自動では使わせない）を強制

既存の npm/pnpm/yarn/bun の設定ファイル（lockfile、`.npmrc`、workspaces、catalogs、overrides/resolutions）を読み取れるため、既存プロジェクトへの導入コストが低い。

### ③ なぜ効果的か（ベンチマーク）

- **スクリプト実行**: `nub run` 14.7ms vs `npm run` 329.9ms（22倍） vs `pnpm run` 442.7ms（30倍）
- **パッケージ実行**: `nubx esbuild --version` 11ms vs `pnpm exec` 191ms（17倍） vs `npx` 226ms（19倍）
- **パッケージインストール**: `nub` 1122ms vs `bun` 1444ms（29%遅い） vs `pnpm` 2847ms（2.5倍） vs `npm` 4163ms（3.7倍）
- Node のバージョンに応じて Temporal、URLPattern、`RegExp.escape`、`Error.isError`、`Promise.try`、`Float16Array`、`navigator.locks`、`vm.Module`、`ShadowRealm`、Wasm imports、WebSocket、EventSource、`node:sqlite`、addon imports などモダン API を自動でポリフィル/フラグ解除
- 単一 Rust バイナリのため起動オーバーヘッドが小さく、上記ベンチマークの速度差に直結する

Colin McDonnell 自身の X 投稿によれば、Nub は彼にとって GitHub スター 1,000 を超えた 5 個目の OSS プロジェクトとなった。Hacker News の "Show HN" スレッド（https://news.ycombinator.com/item?id=48660267）でも活発に議論されている。

---

## 2. Node.js セキュリティリリース（2026年6月18日）

**原文リンク:** [June 2026 Security Releases](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases)

Node.js 22.23.0 / 24.17.0 / 26.3.1 が同時リリースされ、公式ページでは合計 **13件の CVE** が修正された（Node Weekly本文の見出しでは「11 vulnerabilities」と表記されているが、Node.js公式のセキュリティ告知ページを実際に確認したところ合計は13件だった。件数の差はNode Weekly側の要約基準によるものと考えられる）。

### ① 目的・問題意識

TLS/HTTP2/Permission Model など、リモートから悪用可能な脆弱性を含む定期セキュリティパッチ。特に高深刻度の2件はDoSや認証バイパスに直結するため全ラインで即時更新が推奨される。

### ② 具体的な技術詳細

#### 高深刻度 (2件)

| CVE | 内容 | 影響バージョン |
|-----|------|---------------|
| CVE-2026-48933 | WebCrypto AES 整数オーバーフロー。`subtle.encrypt()` への入力サイズが 2GiB の倍数だとプロセスがクラッシュ（リモートDoS） | 22, 24, 26 |
| CVE-2026-48618 | TLS ワイルドカード深度の認証バイパス。Unicode のドット区切り文字の扱いにより、ホスト名の正規化がリゾルバと検証側で不一致になる | 22, 24, 26 |

#### 中深刻度 (6件)

| CVE | 内容 | 影響バージョン |
|-----|------|---------------|
| CVE-2026-48615 | プロキシ認証情報が `ERR_PROXY_TUNNEL` のエラーメッセージ経由で漏洩（ログ等に露出しうる） | 22, 24, 26 |
| CVE-2026-48619 | `node:http2` クライアントで攻撃者が制御する ORIGIN フレームにより無制限にメモリが増加（OOM） | 22, 24, 26 |
| CVE-2026-48937 | HTTP/2 の GOAWAY 送信後のクリーンアップ失敗。プロトコルエラー後もデータを受け付けてしまう | 22, 24 |
| CVE-2026-48928 | mTLS 認可バイパス。大文字小文字を区別するホスト名マッチングにより信頼ポリシーを回避可能 | 22, 24, 26 |
| CVE-2026-48930 | ホスト名に埋め込まれた NUL 文字によるオーソリティ再バインド（リゾルバのバインディングでの C 文字列切り捨て） | 22, 24, 26 |
| CVE-2026-48934 | 異なる servername での TLS セッション再利用により証明書検証をバイパス | 22, 24, 26 |

#### 低深刻度 (Permission Model 関連、`--permission` フラグ利用時のみ影響)

```javascript
// CVE-2026-48617: process.report.writeReport() のパス検証不備で
// パーミッションモデルをバイパス可能

// CVE-2026-48935: FileHandle.utimes() (promises API) が
// 読み取り専用パスのファイルメタデータ変更を許してしまう

// CVE-2026-48936 (Node.js 26のみ): Unix ドメインソケットサーバーが
// --allow-net 権限なしで起動できてしまう

// CVE-2026-48931: http.Agent のレスポンスキュー汚染
// (TOCTOU レースコンディション)
```

#### 同時アップデートされた依存関係

| パッケージ | バージョン |
|-----------|-----------|
| llhttp | 9.4.2 |
| nghttp2 | 1.69.0 |
| OpenSSL | 3.5.7 |
| undici | 8.5.0 (26.3.1) / 7.28.0 (24.17.0) / 6.27.0 (22.23.0) |

### ③ なぜ効果的か

- 高深刻度の2件は**リモートから無認証でトリガー可能**（DoS・認証バイパス）であり、TLS終端やHTTP/2を使う本番サービスは即時パッチが必須
- 3つのアクティブラインすべてに同時対応することで、どのLTSを使っていても同じ日にパッチを適用できる
- Permission Model（実験的機能）関連のバイパスも合わせて塞ぐことで、サンドボックス用途での信頼性が向上

---

## 3. Babel 8.0.0 リリース

**原文リンク:** [Releasing Babel 8 today](https://babeljs.io/blog/2026/06/16/8.0.0/) / [GitHub Discussion #18072](https://github.com/babel/babel/discussions/18072)

2026年6月16日リリース。

### ① 目的・問題意識

Babel 7 は8年間使われ続け、その間に週間 npm ダウンロード数は **170万 → 6億5,100万** にまで成長した。Babel 8 は新しい変換機能を追加する目的ではなく、**次の8年間を見据えた基盤の刷新**（ESM化・デフォルト設定の現代化）が目的。

### ② 具体的な技術詳細（破壊的変更）

1. **ESM専用化**: Babel自体がESMとして配布され、Node.js 22/24/26以上が必須になった。すべての現行LTSラインで `require(esm)` が安定化したことで、CommonJSの利用者もESM版Babelを引き続き `require()` できる。
2. **デフォルトでES5/CommonJSへコンパイルしなくなった**: `@babel/preset-env` のデフォルトターゲットがES5からBrowserslistの `defaults` クエリ（おおよそES2023相当）に変更され、出力形式もデフォルトでCommonJSではなくESMになった。
3. **全Babelパッケージに型定義を同梱**: TypeScriptでBabelのプログラマティックAPIを使う際に `@types/babel__*` 系パッケージが不要になった。
4. **「移動するターゲット」型のブラウザ対応**: evergreenブラウザに機能が広く行き渡るにつれ、Babelは自動的にその機能の変換を止めるようになり、出力バンドルサイズが時間とともに縮小する。
5. その他: モジュールのデフォルトが「ESMサポートを前提とする」`modules: auto` に変更（babel-cli/preset-env/register横断）、`@babel/runtime-corejs3` 用のcorejsエクスポートを削除、パーサーオプション `locations: "packed"` を撤廃。

#### 移行の進め方

まずBabel 7のままで新しいデフォルト設定（target/moduleの明示指定）に合わせて設定を整え、その後にBabel 8へバージョンアップする、という**設定調整とバージョンアップを同時にしない**進め方が公式に推奨されている。移行ガイドは利用者向け（v8-migration）とプラグイン作者向け（v8-migration-api）の2種類が用意されている。

#### サポート期間

Babel 7は**2027年6月まで**セキュリティ修正のみ提供され、新機能のバックポートは行われない。

### ③ なぜ効果的か

- ESM専用化により、Babel内部のコードベースが簡素化されメンテナンス負荷が下がる一方、`require(esm)` の安定化のおかげで既存CommonJSユーザーへの影響を最小化できている
- デフォルトでモダンな出力になることで、多くのプロジェクトで**追加設定なしにより小さく高速なコード**が生成されるようになる
- 型定義の同梱により、TypeScriptプロジェクトでの**依存関係管理が単純化**される

---

## 4. 求人オファーに偽装したバックドア — npmサプライチェーン攻撃 (Foxtopia / pretie_x1)

**原文リンク:** [Fake job offer npm supply chain malware: Foxtopia](https://aydinnyunus.github.io/2026/06/22/fake-job-offer-npm-supply-chain-malware-foxtopia/) / [OSV: MAL-2026-5919](https://osv.dev/vulnerability/MAL-2026-5919)

セキュリティ研究者 Yunus Aydın によるインシデントレポート。

### ① 目的・問題意識

ソーシャルエンジニアリングとサプライチェーン攻撃を組み合わせた事例。著者は「Adam Q」を名乗る人物から、NFTステーキングのフロントエンド案件 "Foxtopia" を7,500ドルで完成させてほしいというメールを受け取り、Googleドライブ経由で「既存」のソースコードを渡された。

### ② 具体的な技術詳細

#### 仕込まれた罠

プロジェクトの `package.json` の devDependencies に、`prettier` にそっくりなタイポスクワッティングパッケージ **`pretie_x1`** が仕込まれていた。ダウンロード数がほぼゼロの無名パブリッシャーであるにもかかわらず、"Opinionated code formatter..." というprettierそっくりのdescriptionとキーワードを持ちながら、実際のフォーマッタ機能は一切含んでいなかった。

#### マルウェアの動作

```json
{
  "scripts": {
    "install": "node cli.js"
  }
}
```

- `npm install` 実行時に発火する **postinstallフック**がエントリーポイント
- `cli.js` が `lib/mirror.js` 内の `resolveConfig() → scheduleMirrorRefresh()` を呼び出す
- 2つのURL（`https://api.aavcareer.ink/installguardd.js` とフォールバック先 `https://hiring.aavcareer.ink/installguardd.js`）が**base64でハードコード**されている
- ペイロードは `os.tmpdir()/bsl-<pid>.js` にダウンロードされるが、その際 **TLS証明書検証を無効化**（`rejectUnauthorized: false`）
- ダウンロード後、**最小サイズ（1024バイト）チェック**と**JSであってHTMLでないことの確認**（ハニーポット対策）を行ってから実行
- ペイロードは `process.execPath` 経由で**デタッチ・隠蔽された状態**で起動
- **回避策**: CI環境を検知すると実行をスキップ（自動マルウェアスキャンを回避）。実際のマルウェア本体は**AES-256-GCMで暗号化**されており平文でディスクに書かれない。C2のローテーションにはGitLabの「デッドドロップ」を利用

#### ペイロードの機能

約557KBの情報窃取ツールで、ブラウザに保存されたパスワード、暗号資産ウォレット、SSH鍵、`.env` ファイルを収集し、さらに**偽のOSログインダイアログ**を表示してOSパスワードまでフィッシングする。

#### 検知

Aikido Intel の自動脅威検知チームが、`pretie_x1` の公開された全5バージョン（3.8.5〜3.8.9）を**公開から数分以内**にマルウェアと判定した。OSVアドバイザリ **MAL-2026-5919** として追跡されている。

### ③ なぜ効果的か（攻撃者視点）／教訓

- タイポスクワッティング名 + それっぽいdescriptionにより、**手作業でのpackage.jsonレビューでは見逃されやすい**
- postinstallフックの悪用は**依存関係を追加するだけで実行される**ため被害者の能動的操作が不要
- CI検知によるスキップやAES暗号化によって、**静的スキャンや自動サンドボックス解析からの回避**を狙っている
- 「未払い報酬をエサにした求人詐欺」「前任者が失踪したという作り話」「使い捨てメールドメイン」といった**社会工学的な違和感のサイン**を認識することが、自動検知に頼らない第一の防御線になる

---

## 5. Node.js 26 で見落としがちな新機能まとめ

**原文リンク:** [What's new in Node.js 26](https://nodejsdesignpatterns.com/blog/whats-new-in-nodejs-26/)

Node.js 26.0.0 は2026年5月5日リリース。本記事は実務で見落とされがちな新機能を実践的なコード付きで紹介する記事。

### ① 目的・問題意識

メジャーリリースのアナウンスは大きな目玉機能（Temporalなど）に注目が集まりがちだが、日常のコーディングを地味に改善する小粒な追加API（Map/WeakMapのユーティリティ、Iteratorの合成など）は見落とされやすい。それらを実践コード付きで棚卸しする。

### ② 具体的なコードと技術詳細

#### Temporal API（安定化・グローバル化）

Node.js 26でTemporalがグローバルオブジェクトとして安定版になった。ミュータブルでタイムゾーンの扱いが不透明だった従来の `Date` を置き換える、イミュータブルでタイムゾーン対応の日時APIである（詳細は本ニュースレターの過去号でも既出）。

#### `Map.prototype.getOrInsert` / `getOrInsertComputed`（`WeakMap` にも同様に追加）

「キーがあれば取得、なければ値をセットして返す」という定型処理（`if (!map.has(k)) map.set(k, v)`）を1メソッドに置き換える。

```javascript
const wm = new WeakMap();
const obj = {};

console.log(wm.get(obj)); // undefined
console.log(wm.getOrInsert(obj, "default")); // "default" (挿入して返す)
console.log(wm.get(obj)); // "default"
console.log(wm.getOrInsert(obj, "another default")); // "default" (既にあるので上書きしない)
```

`getOrInsert` は値をそのまま渡す（生成コストが低いデフォルト値向け）。`getOrInsertComputed` はファクトリ関数を渡す形で、生成コストの高いデフォルト値（重い初期化を伴うオブジェクトなど）を必要な時だけ計算するのに使う。他人が所有するオブジェクトにメタデータを紐付けたい場合（`WeakMap` 版）に特に有用。

#### `Iterator.concat()`

複数のイテラブルを、中間配列を作らずに**遅延評価で連結**する静的メソッド。

```javascript
const map1 = new Map([["a", 1], ["b", 2]]);
const map2 = new Map([["c", 3], ["d", 4]]);
const map3 = new Map([["a", 5], ["e", 6]]);

const map = new Map(Iterator.concat(map1, map2, map3));
```

`Array.prototype.concat()` の考え方をあらゆるイテラブルに一般化したもので、配列ではなくイテレータを返す点が異なる。イテラブルでない素のイテレータを渡す場合は事前に `Iterator.from()` でラップする必要がある。

#### crypto: Ed25519 の生鍵 (raw key) サポートと署名の "context" オプション

生のEd25519鍵バイト列をエクスポート・利用できるようになった。

```javascript
// 生の鍵バイト列としてエクスポート
const rawPrivate = generated.privateKey.export({ format: 'raw-private' });
const rawPublic = generated.publicKey.export({ format: 'raw-public' });

// 署名・検証は引き続き sign(null, data, privateKey) / verify(null, data, publicKey, signature)
```

署名APIに "context" パラメータが追加され、同じ鍵ペアでも用途ごとに異なるコンテキスト文字列を紐付けた署名を作れるようになった（用途の異なる署名を混同・再利用されるリスクを減らす）。

#### V8 14.6 / Undici 8

V8のバージョンアップとUndici（`fetch()` の内部実装）のメジャーアップデートにより、全体的なランタイム・HTTPクライアント性能が向上。

#### 非推奨・レガシーAPIの整理

長年非推奨だったAPI群がまとめて整理された。ほとんどのアプリには影響しない見込みだが、CIのNodeバージョンを上げる前に一度スキャンしておく価値があるとされている。

### ③ なぜ効果的か

- `getOrInsert`/`getOrInsertComputed` により、Map/WeakMapの「あれば取得・なければ挿入」という頻出パターンのボイラープレートと**二重ルックアップ**が解消される
- `Iterator.concat()` は複数コレクションを結合する際に**中間配列を作らない**ため、大きなデータセットに対してメモリ効率がよい
- Ed25519の生鍵サポートは鍵フォーマットの相互運用性（他言語・他システムとの鍵共有）を高め、署名の"context"はキー使い回しに伴う誤用リスクを下げる
- 小粒な機能ではあるが、いずれも**外部ライブラリなしで済ませられる**という共通の効果がある

---

## まとめ

| トピック | 重要度 | アクション |
|---------|--------|-----------|
| Nub（Node拡張ツールキット） | ★★★ | `npm run`/`npx`/`nvm` の起動が遅いと感じるなら試す価値あり |
| Node.js セキュリティリリース（13 CVE） | ★★★ | 22.23.0 / 24.17.0 / 26.3.1 へ**今すぐ**アップデートする |
| Babel 8.0.0 | ★★☆ | Node 22+ が前提になった点とデフォルト出力のESM化を確認してから移行 |
| npmサプライチェーン攻撃（求人詐欺型） | ★★★ | 未知のdevDependencyやpostinstallスクリプトを安易に信用しない、社会工学的な違和感に注意 |
| Node.js 26 の見落としがちな新機能 | ★★☆ | `Map.getOrInsert`・`Iterator.concat` など地味に便利なAPIを活用する |
