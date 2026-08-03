# Node Weekly Issue 635 まとめ (July 30, 2026)

> 出典: [Node Weekly Issue 635](https://nodeweekly.com/issues/635)

---

## 1. npm、公開時（publish-time）マルウェアスキャンと「デュアルユース」メタデータを導入

**原文リンク:** [npm publish-time malware scanning and dual-use metadata (GitHub Changelog)](https://github.blog/changelog/2026-07-28-npm-publish-time-malware-scanning-and-dual-use-metadata/)

### ① 目的・問題意識

npm レジストリはサプライチェーン攻撃（悪意あるパッケージの公開、既存パッケージの乗っ取りなど）の標的になり続けている。従来は公開後にマルウェアが検知されてから削除する「事後対応」が中心だったため、悪意あるパッケージが一時的にでも `npm install` 可能な状態になってしまう問題があった。

一方で、セキュリティ関連ツール（ポートスキャナ、資格情報チェッカー、ペネトレーションテストツールなど）は正規の目的で低レベル・侵襲的な挙動を行うため、自動スキャナからは「悪意あるコード」と誤検知されやすい。この「デュアルユース（両用）」パッケージを機械的なブロックから救済しつつ、悪意あるパッケージは確実に止める仕組みが必要だった。

### ② 具体的な技術詳細

**公開時スキャンのフロー**

- 新しく公開されたパッケージは、install 可能になる前に自動スキャンされる。
- スキャン結果に応じて、
  1. 通常どおり公開される
  2. 手動レビューのため保留される
  3. ブロックされる
  のいずれかになる。
- この結果、公開から実際にインストール可能になるまでに遅延が発生する。**通常は約5分程度**だが、ピーク時やパッケージの内容・サイズによっては **15分以上** かかることもある。

**デュアルユースコンテンツの申告メタデータ**

正規のセキュリティツールなどがスキャンに引っかからないよう、`package.json` に新しい `contentPolicy` フィールドが追加された。

```json
{
  "name": "my-security-tool",
  "version": "1.0.0",
  "contentPolicy": {
    "dualUse": true
  }
}
```

このフィールドを宣言する場合、以下のルールが適用される：

- パッケージのルートディレクトリに **`DISCLOSURE`** という名前のファイルを追加する必要がある。
- `DISCLOSURE` ファイルはテキスト内容のみで、そのデュアルユース機能の内容と、意図された正当な用途を説明する自由形式のドキュメントである。
- npm の Trust & Safety チームがこのメタデータをレビュー時に参照する。宣言したからといって自動的に公開が承認されるわけではなく、ケースバイケースでレビューされることがある。
- デュアルユースを宣言したパッケージは、**二要素認証（2FA）を強制する公開方法**（トークンではなく2FA必須の公開経路）を経由しなければならない。
- 一度 `contentPolicy` と `DISCLOSURE` を追加したパッケージは、以降のバージョンでこれらを削除できない。削除しようとした公開は拒否される。

### ③ なぜそれが効果的か

- **「公開してから消す」から「公開前に止める」への転換** により、悪意あるパッケージがエンドユーザーの `node_modules` に一瞬でも到達するリスクを減らす。
- **デュアルユースの申告制度** により、セキュリティ業界のツール開発者を過検知で締め出さずに済み、誤検知によるエコシステムの萎縮を防ぐ。
- **申告メタデータの削除不可・2FA必須化** により、「デュアルユースを詐称して一度承認を得てから悪意あるコードに差し替える」という迂回策を防止できる。
- 5〜15分という遅延は、自動化された大量公開・大量インストールを狙う攻撃（公開直後にワームのように広がるタイプ）の時間的猶予を作り、スキャン・ブロックの実効性を高める。

---

## 2. Node.js、新しい API ドキュメントサイトのベータ版を公開

**原文リンク:** [Check out the New Node.js API Documentation Preview](https://nodejs.org/en/blog/announcements/new-api-docs-beta)

### ① 目的・問題意識

従来の Node.js API ドキュメント（`nodejs.org/docs/`）には、以下のような課題があった。

- ページ内検索機能が組み込まれておらず、検索するには外部の検索エンジンに頼る必要があった。
- ナビゲーションとレイアウトが情報を探しにくい構造になっていた。
- nodejs.org 本体サイトとデザインシステムが統一されていなかった。

これらを解決するため、ドキュメント生成の仕組み自体を作り直した新しい API ドキュメントのベータ版が `beta.docs.nodejs.org` で公開された。

### ② 具体的な技術詳細

**新機能**

- **組み込み検索（初導入）**: 全ページに検索ボックスがあり、`⌘K`（Cmd+K）のキーボードショートカットで即座に検索を開ける。
- **常時表示のサイドバー**: 全モジュールの一覧を表示する永続的なサイドバーナビゲーション。
- **常時表示の目次（TOC）**: 各ページの目次が常に見える状態で表示される。
- **レスポンシブデザイン**: 小さい画面幅でも使えるレイアウト。
- **読了時間の表示**: 各ページに推定読了時間を表示。
- **アナウンスバー**: 重要な通知を表示できるバナー機能。
- **`llms.txt` 対応**: AI ツールが API リファレンスに構造化された形でアクセスできるよう `llms.txt` を提供（`nodejs.org/llms.txt` としてすでに本体サイトには導入済み）。

**維持された既存機能**

- ライト／ダークテーマ
- ESM/CJS のコードスニペット切り替え
- コードスニペットのコピー用ボタン
- Stability（安定度）バッジ
- バージョン履歴
- 各ページの Markdown 版

**アクセス方法・技術実装**

- ベータ版は https://beta.docs.nodejs.org/ からアクセス可能。
- コンテンツ自体は `nodejs/node` リポジトリの既存の Markdown ファイルから生成されており、内容の変更はなく、表現・機能面のみが刷新されている。
- 生成エンジンには Node.js プロジェクトが新たに開発した専用ツール **doc-kit**（https://github.com/nodejs/doc-kit）が使われており、従来のレガシーなドキュメントジェネレータを置き換える。
- フィードバックは `nodejs/doc-kit` の GitHub Issues で受け付けている。
- JavaScript を無効にした状態でも、オフラインでも利用可能な設計になっている。

### ③ なぜそれが効果的か

- **組み込み検索** により、外部検索エンジンへの「回り道」がなくなり、目的の API に到達するまでの手数が減る。
- **既存機能を維持しつつ刷新** することで、既存ユーザーの学習コストや作業フローを壊さない後方互換的なアップグレードになっている。
- **nodejs.org 本体との統一デザイン** により、公式サイト全体の一貫した体験を提供できる。
- **`llms.txt` 対応** は、AI コーディングアシスタントが正確な API 情報を取得しやすくするための布石であり、LLM 時代のドキュメント設計として先進的。
- **JS 不要・オフライン対応** はアクセシビリティと堅牢性を高める設計判断。
- ベータ期間を設けて実運用でのフィードバックを収集してからデフォルト化する、段階的リリース戦略を取っている。

---

## 3. Node.js セキュリティリリース (2026年7月29日) — 11件のCVEを修正

**原文リンク:** [Node.js — July 2026 Security Releases](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases)

### ① 目的・問題意識

Node.js の 22.x / 24.x / 26.x の各リリースラインに影響する、HTTP/2、HTTPS、Permission Model、`node:sqlite`、`node:zlib`、DNS、HTTPパーサなど広範囲のコンポーネントに存在する脆弱性をまとめて修正するための定例セキュリティリリース。重大度は **High 3件・Medium 5件・Low 3件** の計11件。

### ② 具体的な技術詳細（CVE一覧）

**High（重大）**

1. **CVE-2026-56846** — HTTP/2 の保持されたヘッダーが `maxSessionMemory` 制限を回避できる（対象: 24.x, 22.x）。リモートからのメモリ枯渇を引き起こし得る。修正版: v22.23.2, v24.18.1
2. **CVE-2026-56848** — HTTP/2 の再入的な send 呼び出しで heap-use-after-free が発生（対象: 26.x, 24.x, 22.x）。`nghttp2_session_mem_recv()` 実行中に `nghttp2_session_mem_send()` が再入的に呼ばれることが原因。修正版: v26.5.1, v24.18.1, v22.23.2
3. **CVE-2026-58043** — Permission Model のパスマッチングがファイルシステムアクセスを過剰に許可してしまう（対象: 26.x, 24.x, 22.x）。radix-tree のプレフィックス境界処理の不備を突かれ、`--permission` フラグで意図したアローリスト外のパスに読み書きできてしまう。修正版: v26.5.1, v24.18.1, v22.23.2

**Medium（中）**

4. **CVE-2026-56850** — HTTPS Agent が異なる PFX 証明書間で mTLS のクライアント識別情報を再利用してしまう（PFXオブジェクト配列のキー衝突が原因）。修正版: v26.5.1, v24.18.1, v22.23.2
5. **CVE-2026-58040** — HTTPS Agent のセッション再利用がホスト名検証をスキップしてしまう（CVE-2026-48934 の不完全な修正）。修正版: v26.5.1, v24.18.1, v22.23.2
6. **CVE-2026-58041** — `node:sqlite` の `SQLTagStore` イテレータのリプレイにより、書き込みが再実行されてしまう。`DatabaseSync#createTagStore()` で作成された古い `StatementSyncIterator` がパラメータリセット後もキャッシュされたプリペアドステートメントを実行し続けてしまう。修正版: v26.5.1, v24.18.1
7. **CVE-2026-58042** — `dns.resolveAny()` が A レコードを256件超含むDNSレスポンスでアボートしてしまい、繰り返しトリガーするとDoSにつながる。修正版: v26.5.1, v24.18.1, v22.23.2
8. **CVE-2026-58045** — `node:zlib` の同期APIが、偽装された `TypedArray` の長さでクラッシュしてしまう（到達可能なアサーションによるプロセスクラッシュ、DoSにつながる）。修正版: v26.5.1, v24.18.1, v22.23.2

**Low（低）**

9. **CVE-2026-56847** — Permission Model 下で `trace_events.createTracing().enable()` がアローリスト外にトレースログを書き込める。修正版: v26.5.1, v24.18.1, v22.23.2
10. **CVE-2026-58039** — Permission Model 下で `process.report` がアローリスト外のファイルを書き込み・上書きできる。修正版: v26.5.1, v24.18.1, v22.23.2
11. **CVE-2026-58044** — HTTPパーサのヘッダー切り捨てによりリクエストスマグリングが可能になる。`maxHeadersCount`/`maxHeaderPairs` を超えるヘッダーが `req.headers`/`req.rawHeaders`/`req.headersDistinct` からは省かれるが、内部的なフレーミング処理には使われ続けるため、フォワーディングプロキシ環境でクライアントとサーバー間のリクエスト解釈がずれる（`Content-Length` を隠しながらボディを配送できてしまう）。修正版: v26.5.1, v24.18.1, v22.23.2

**依存ライブラリの更新**

- `undici`: 8.9.0 (26.x) / 7.29.0 (24.x) / 6.28.0 (22.x)
- `llhttp`: 9.4.3（全ライン共通）

**推奨アップグレード先**

| リリースライン | アップグレード先 |
|---|---|
| 26.x | v26.5.1 |
| 24.x | v24.18.1 |
| 22.x | v22.23.2 |

### ③ なぜそれが効果的か

- CVE-2026-58043・56847・58039 はいずれも **Permission Model（`--permission`）の境界を破る系統の脆弱性** であり、「制限付きで実行しているつもり」が実際には制限が効いていなかったことを意味する。パーミッションモデルを信頼して機密ファイルアクセス制御に使っている運用ほど影響が大きいため、優先度高く適用すべき。
- CVE-2026-58044（リクエストスマグリング系）は、リバースプロキシ経由の本番環境で **HTTPリクエストの密輸攻撃** に直結しうるため、プロキシ構成を取っているサービスは特に注意が必要。
- CVE-2026-58040 は既存の修正（CVE-2026-48934）が不十分だったことによる再発であり、TLS 関連の修正は再検証が重要であることを示す一例。
- 全体として、3ライン（22.x/24.x/26.x）同時対応のため、LTSも含め幅広い利用者に影響がある。速やかなアップグレードが推奨される。

---

## 4. 2026年における「安全な npm パッケージの公開方法」

**原文リンク:** [The secure way to release an npm package in 2026 (Evil Martians)](https://evilmartians.com/chronicles/the-secure-way-to-release-an-npm-package)

### ① 目的・問題意識

従来の npm パッケージ公開フローは、開発者のローカル環境や CI に **長期有効な npm トークン（`NPM_TOKEN`）** を保存し、それを使って `npm publish` するのが一般的だった。しかし長期トークンには次のリスクがある。

- CI のログや設定ファイルに誤って露出しうる
- 手動でのローテーション・管理が必要
- 一度漏洩すると、失効させるまで永続的に悪用され続けられる

この記事は、「盗まれるトークンがそもそも存在しない」公開フロー——**Trusted Publishing（信頼された公開）・Provenance（来歴証明）・Staged Publishing（段階的公開）** を GitHub Actions 1本のワークフローで組み合わせる方法を解説している。

### ② 具体的な技術詳細

**Trusted Publishing（OIDC 認証）**

OpenID Connect (OIDC) を使い、GitHub Actions・GitLab CI/CD・CircleCI と npm レジストリの間に信頼関係を設定することで、長期トークンなしにパッケージを公開できる。OpenSSF（Open Source Security Foundation）が定める Trusted Publishers 業界標準に準拠。

- 要件: npm CLI v11.5.1 以降、Node.js v22.14.0 以降
- npm CLI は OIDC 環境を自動検出し、トークン認証よりも優先して使用する
- セルフホストランナーは現時点では未対応（GitHub-hosted / GitLab.com shared runner / CircleCI cloud のみ）

npmjs.com 側の設定（パッケージ設定 → Trusted Publisher）で、GitHub Actions の場合は以下を登録する：

- Organization/user 名（必須）
- リポジトリ名（必須）
- ワークフローファイル名（必須、`.yml`/`.yaml` 拡張子込みの完全一致・大小文字区別あり）
- Environment 名（任意）
- 許可するアクション（`npm publish` / `npm stage publish` のいずれか、または両方）

**GitHub Actions ワークフロー例**

```yaml
name: Publish Package

on:
  push:
    tags:
      - 'v*'

permissions:
  id-token: write  # OIDCトークン発行に必須
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v6
        with:
          node-version: '24'
          registry-url: 'https://registry.npmjs.org'
          package-manager-cache: false  # リリースビルドではキャッシュを使わない
      - run: npm ci
      - run: npm run build --if-present
      - run: npm test
      - run: npm publish  # または: npm stage publish
```

`permissions.id-token: write` が OIDC トークン発行の鍵であり、これが無いと認証できない。`workflow_call`/`workflow_dispatch` を使う場合は、呼び出し元・呼び出し先の両方のワークフローにこの権限が必要。

**GitLab CI/CD の例**

```yaml
stages:
  - test
  - build
  - publish

variables:
  NODE_VERSION: '24'

test:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm test

publish:
  stage: publish
  image: node:${NODE_VERSION}
  id_tokens:
    NPM_ID_TOKEN:
      aud: "npm:registry.npmjs.org"
    SIGSTORE_ID_TOKEN:
      aud: sigstore
  script:
    - npm ci
    - npm run build --if-present
    - npm publish  # または: npm stage publish
  only:
    - tags
```

**CircleCI の例**

```yaml
version: 2.1

jobs:
  publish:
    docker:
      - image: cimg/node:22.14
    steps:
      - checkout
      - run:
          name: Install dependencies
          command: npm ci
      - run:
          name: Run tests
          command: npm test
      - run:
          name: Build
          command: npm run build --if-present
      - run:
          name: Publish to npm with OIDC
          command: |
            export NPM_ID_TOKEN=$(circleci run oidc get --claims '{"aud": "npm:registry.npmjs.org"}')
            npm publish  # または: npm stage publish

workflows:
  publish:
    jobs:
      - publish:
          filters:
            tags:
              only: /^v.*/
            branches:
              ignore: /.*/
```

**自動 Provenance（来歴証明）生成**

Trusted Publishing 経由で GitHub Actions / GitLab CI/CD から公開すると、`--provenance` フラグを付けなくても自動的に provenance attestation（そのパッケージがどのソースコード・どのワークフロー実行から作られたかを暗号学的に証明する情報）が生成・公開される。ただし以下の条件をすべて満たす必要がある：

- Trusted Publishing（OIDC）経由での公開であること
- 公開リポジトリからの公開であること
- 公開パッケージであること（プライベートパッケージには provenance は生成されない。プライベートリポジトリの場合も同様）
- CircleCI では provenance 生成は未対応

Provenance を無効化したい場合：

```bash
# 環境変数で無効化
NPM_CONFIG_PROVENANCE=false npm publish
```

```ini
# .npmrc で無効化
provenance=false
```

```json
// package.json で無効化
{
  "publishConfig": {
    "provenance": false
  }
}
```

**Staged Publishing（段階的公開）+ ハードウェア2FA承認**

記事のポイントの一つは、CI 公開とハードウェアの2FAトークンによる手動承認を組み合わせる「Staged Publishing」。CI は `npm stage publish` でパッケージをステージング（＝まだ一般公開されない保留状態）し、メンテナが手動でハードウェアの2FAトークンを使って承認して初めて公開が完了する。これにより、**たとえCI環境が乗っ取られても、攻撃者は人間の承認なしにパッケージを公開できない**。

これを実現するための npmjs.com 側の設定手順：

1. パッケージ設定 → Publishing access → **「Require two-factor authentication and disallow tokens」** を選択
2. Trusted Publisher の「許可するアクション」を **`npm stage publish` のみ**（`npm publish` は許可しない）に限定
3. これにより CI からの自動公開はすべて staged（保留）状態になり、人間が2FAで最終承認するまで一般公開されない

移行の際の推奨手順：

1. まず Trusted Publisher を設定し、動作確認する
2. その後にトークン認証を禁止する設定へ切り替える
3. 不要になった既存の自動化用トークンを失効させる

**プライベート依存関係のインストール**

Trusted Publishing は `npm publish` コマンドのみに適用されるため、プライベートパッケージへの依存がある場合は、インストール用に別途 read-only の granular access token が必要になる。

```yaml
- uses: actions/setup-node@v6
  with:
    node-version: '24'
    registry-url: 'https://registry.npmjs.org'
    package-manager-cache: false
# 依存関係インストールには read-only トークンを使用
- run: npm ci
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_READ_TOKEN }}

# 公開自体はOIDC経由（トークン不要）
- run: npm publish
```

### ③ なぜそれが効果的か

- **盗めるトークンがそもそも存在しない**: OIDC による短命でワークフロー固有のクレデンシャルは、漏洩・誤コミット・ログ露出のリスクを構造的になくす。トークンのローテーション運用も不要になる。
- **Provenance によるサプライチェーンの可視性**: パッケージがどのコミット・どのCI実行から生成されたかを暗号学的に検証可能にすることで、「正規のソースから公開されたか」をエンドユーザー・依存関係管理ツール側で検証できるようになる。
- **Staged Publishing + ハードウェア2FA** が最大の防御層: CI/CDパイプライン自体が侵害されるサプライチェーン攻撃（近年頻発している GitHub Actions ワークフロー改ざんや依存パッケージ経由の侵害）に対しても、「人間による物理トークンでの最終承認」という単一障害点にならない防御を追加できる。CIの自動化とセキュリティを両立させる設計。
- **段階的移行が可能**: 「まずTrusted Publisherを設定→動作確認→トークンを禁止」という順序を踏むことで、既存の公開フローを壊さずに安全な方式へ移行できる。

---
