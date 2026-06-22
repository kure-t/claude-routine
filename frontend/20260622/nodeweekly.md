# Node Weekly Issue 629 (June 18, 2026)

> 最新号: https://nodeweekly.com/issues/629

---

## 1. Nub: Node.jsをAugmentするオールインワンツールキット

**リンク:** https://nubjs.com/blog/introducing-nub  
**GitHub:** https://github.com/nubjs/nub

### ① 目的・問題意識

Bun や Deno が Node.js を「置き換えよう」とするアプローチに対し、Nub は「既存の Node.js を拡張する」立場を取る。Zod・Standard Schema・tRPC の作者である Colin McDonnell（元 Bun エンジニア）が開発。TypeScript ファイルをそのまま実行できない、npm run/npx が遅い、パッケージインストールに postinstall スクリプトが無防備に走るといった Node.js の痛点を、ランタイム移行なしに解消する。

### ② 具体的な技術詳細・コード

**インストール:**
```bash
npm install -g --ignore-scripts=false @nubjs/nub
```

**主要コマンド:**
```bash
# TypeScript ファイルを直接実行
nub index.ts
nub src/server.ts

# ファイル変更時に自動再起動
nub watch src/server.ts

# package.json スクリプトを実行（pnpm run の24倍速）
nub run dev
nub run build

# node_modules/.bin のCLIを実行（npx の19倍速）
nubx prisma generate
nubx tsc --noEmit

# 依存パッケージをインストール（pnpm互換）
nub install

# パッケージマネージャーの shim を設定（Corepak スタイル）
nub pm shim

# Node.js バージョン管理
nub node install 26
nub node use 22
```

**TypeScript サポートの範囲:**
```typescript
// 通常の型消去に加え、以下の non-erasable 構文もサポート
enum Direction { Up, Down, Left, Right }

// legacy デコレーター（emitDecoratorMetadata 含む）
@Injectable()
class UserService { ... }

// namespace
namespace MyApp {
  export interface Config { ... }
}

// parameter properties
class User {
  constructor(public name: string, private age: number) {}
}

// ES2022 using 宣言
await using db = await openDatabase();
```

**パッケージマネージャーの特徴:**
- pnpm 互換の CLI（pnpm のスペルをそのまま使用可能）
- ロックファイルはプロジェクト既存形式のまま維持（npm/pnpm/Bun のロックファイルを round-trip）
- `lifecycle scripts` はデフォルト deny（postinstall 攻撃への対策）

```bash
# セキュアなインストール：postinstall は明示的に承認が必要
nub install
# 特定パッケージの postinstall を承認する場合
nub install --allow-scripts lodash
```

**Node バージョン管理:**
```bash
# .node-version / .nvmrc / engines.devEngines を自動読み取り
# 対象バージョンが未インストールなら nodejs.org から自動取得・チェックサム検証
nub node install 26.3.1
```

**技術的実装:**
- Rust 製 CLI（起動数ミリ秒）
- TypeScript トランスパイルには Oxc（Rust ネイティブ）を Node addon として利用
- Node 22.15+ ではシンク `module.registerHooks()` の速いパスを使用
- Node 18.19〜22.14 は非同期 `module.register()` のローダーワーカーで互換対応

**パフォーマンス比較:**
| 操作 | Nub | 従来 |
|------|-----|------|
| `nub run` vs `pnpm run` | 24× 速 | - |
| `nubx` vs `npx` | 19× 速 | - |
| `.ts` 起動 vs `tsx` | 2.9× 速 | - |

**動作要件:** Node 18.19+、macOS (arm64/x64)、Linux (x64/arm64)、Windows (x64/arm64)

### ③ なぜ効果的か

既存の Node.js バイナリに乗る設計のため、ランタイム移行ゼロでメリットを享受できる。Rust CLI の起動コストは「数ミリ秒」であり、npm run のような JS 起動オーバーヘッドが排除される。Oxc による in-memory トランスパイルにより、TypeScript が「そのまま動く」体験を、型チェックの速度に依存せず実現。postinstall スクリプトのデフォルト拒否は、サプライチェーン攻撃の主要経路を構造的に塞ぐ。

---

## 2. Node.js June 18, 2026 セキュリティリリース（12 CVEs）

**リンク:** https://nodejs.org/en/blog/vulnerability/june-2026-security-releases  
**解説記事:** https://www.digitalapplied.com/blog/nodejs-june-2026-security-releases-cve-patch-guide

### ① 目的・問題意識

Node.js の全サポートラインに対し、同時にセキュリティパッチを適用。12 件の CVE のうち HIGH が 2 件。Node 20 は 2026 年 4 月 30 日 EOL のため、今回のパッチ対象外（本番で Node 20 を使い続けている組織には公式の緩和手段がない）。

**リリースバージョン:**
- Node.js 22 → 22.23.0 (codename: Jod)
- Node.js 24 → 24.17.0 (codename: Krypton)
- Node.js 26 → 26.3.1 (Current)

**共通バンドル依存バージョンアップ:**
- llhttp → 9.4.2
- nghttp2 → 1.69.0（SEMVER-MAJOR）
- OpenSSL → 3.5.7

### ② 具体的な技術詳細・CVE 一覧

**HIGH severity（2件）:**

**CVE-2026-48933 — WebCrypto 2GiB 整数オーバーフロー（クラッシュ）**
```javascript
// 脆弱なコード例：2GiB の倍数の入力でプロセスがクラッシュ
const key = await crypto.subtle.generateKey({ name: 'AES-GCM', length: 256 }, true, ['encrypt']);
const largeData = new Uint8Array(2 * 1024 * 1024 * 1024); // 2GiB
await crypto.subtle.encrypt({ name: 'AES-GCM', iv: new Uint8Array(12) }, key, largeData);
// ^ プロセスが abort する（意図的な攻撃でなくとも誤って発生しうる）
```
- **影響:** サービスクラッシュ（DoS）
- **対策:** 22.23.0 / 24.17.0 / 26.3.1 へアップデート

**CVE-2026-48618 — TLS ワイルドカード証明書検証バイパス**
```
証明書: *.example.com
正規ホスト: api.example.com ✓
攻撃者が利用するホスト: API.Example.COM（大文字正規化の問題）→ 検証をバイパス可能
```
- **影響:** マルチテナント SaaS 等で最も一般的なワイルドカード証明書パターンを攻撃者が悪用可能
- **対策:** ホスト名正規化ロジックの修正

**MEDIUM/LOW severity（10件）:**

| CVE | 概要 |
|-----|------|
| CVE-2026-48928 | 大文字ホスト名でのSNIマッチングによりmTLS認可バイパス |
| CVE-2026-48930 | ホスト名へのnull byte埋め込みによるCリゾルバ内でのサイレント切り詰め |
| CVE-2026-48934 | 異なるサーバー名でのTLSセッション再利用によるホスト同一性検証バイパス |
| CVE-2026-48615 | エラーメッセージへのプロキシ認証情報漏洩 |
| CVE-2026-48617 | `process.report.writeReport()` 経由の Permission Model バイパス |
| CVE-2026-48619 | HTTP/2クライアントの無制限メモリ増大 |

### ③ なぜ効果的か（対応の優先度）

CVE-2026-48933 の WebCrypto クラッシュは「意図しない操作でも発生しうる」ため、ネットワーク越しに DoS が成立する危険性が高い。CVE-2026-48618 の TLS バイパスはマルチテナント SaaS の主要証明書パターンを直撃する。担当チームごとにグルーピングされた CVE 分析（TLS 担当 4 件、HTTP 担当 4 件、Crypto 1 件、Permission Model 3 件）が運用対応を効率化する。

---

## 3. TypeScript 7.0 RC リリース — Go 製コンパイラで10倍高速化

**リンク:** https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/  
**解説:** https://www.digitalapplied.com/blog/typescript-7-0-rc-go-native-compiler-2026-upgrade-guide

### ① 目的・問題意識

TypeScript のコンパイラを TypeScript（JavaScript）から Go にネイティブ移植（"Project Corsa"）し、ネイティブコードの速度と共有メモリ並列化により大規模コードベースの型チェック・ビルド時間を劇的に短縮する。1.5 百万行規模の VS Code コードベースで 77 秒 → 7.5 秒（約 10 倍）を実現。

### ② 具体的な技術詳細・コード

**インストール（RC）:**
```bash
npm install -D typescript@rc
npx tsc --version
# TypeScript 7.0.0-rc.1
```

**パフォーマンス比較:**
| コードベース | TypeScript 6.0 | TypeScript 7.0 RC |
|-------------|---------------|-------------------|
| VS Code（1.5M行） | 77秒 | 7.5秒（~10倍） |
| 一般的な大規模 monorepo | - | 約10倍高速 |

**技術的実装:**
- TypeScript 6.0 のコードベースを Go に機械的移植（スクラッチ書き直しではない）
- 型チェックロジックは TypeScript 6.0 と構造的に同一
- ネイティブコード実行 + Go の共有メモリ並列化を活用
- `tsc` CLI は引き続き同じインターフェイス

**移行上の注意点:**
```bash
# tsconfig.json の互換性
# TypeScript 7.0 RC は既存の tsconfig.json をそのまま読み込める
# 型チェック結果は 6.0 と同一のはず

# エディタ対応
# VS Code では TypeScript 7.0 RC をワークスペースの TypeScript として指定
# settings.json:
# "typescript.tsdk": "node_modules/typescript/lib"
```

### ③ なぜ効果的か

Go への移植は「同じロジックをネイティブで走らせる」アプローチのため、型システムの互換性を壊さずに速度を得られる。RC は百万行規模のコードベースでの実運用前評価が推奨され、GA は RC から約 1 ヶ月後の予定。大規模モノレポや IDE のレスポンスタイム改善の恩恵が特に大きい。

---

## 4. Node.js 26.3.0 リリース — Buffer高速化・Permission Model強化

**リンク:** https://nodejs.org/en/blog/release/v26.3.0  
**GitHub リリース:** https://github.com/nodejs/node/releases/tag/v26.3.0

### ① 目的・問題意識

Node.js 26 系 Current ラインの定期リリース（2026-06-01）。高I/O環境でのメモリ効率改善、Permission Model の細粒度制御追加、macOS 向け将来変更の告知が含まれる。

### ② 具体的な技術詳細・コード

**`Buffer.poolSize` デフォルト変更: 32 KiB → 64 KiB**
```javascript
// 変更前（Node 26.2.x まで）
Buffer.poolSize; // 32768 (32 KiB)

// 変更後（Node 26.3.0 から）
Buffer.poolSize; // 65536 (64 KiB)

// 意味：小さなバッファを大量に作成する際の内部アロケーション回数が減少
// → 重いI/O処理（ファイルストリーミング等）でのオーバーヘッド削減
```

**`permission.drop()` API 追加**
```javascript
// Node.js の Permission Model に権限を動的に破棄するAPIが追加
// 起動後に不要な権限を明示的に剥奪できる

// 例：初期化後はファイル書き込み権限を破棄
import { permission } from 'node:process';

// 初期化処理（読み書き必要）
await setup();

// 以降は読み取り専用にロックダウン
process.permission.drop('fs.write');

// 以後、fs.writeFile 等はエラーになる（最小権限の原則）
```

**その他の変更:**
```javascript
// HTTP ヘッダー値検証の設定オプション追加
const server = http.createServer({ httpValidation: true }, handler);

// Inspector で正確なカバレッジ開始位置を JS ランタイムに公開
// ルート証明書を NSS 3.123.1 に更新
// QUIC: レート制限・ホスト名検証・ストリームアイドルタイムアウト・ブロックリスト対応
```

**macOS 注意事項:**
- 将来のリリースで macOS Universal バイナリ（arm64 + x64 同梱）の提供を廃止予定
- arm64 / x64 個別バイナリへの移行が必要

### ③ なぜ効果的か

Buffer.poolSize 倍増は既存コードの変更なしにI/Oヘビーなアプリのメモリ効率が向上する。`permission.drop()` は「初期化フェーズのみ広い権限 → 実行フェーズは最小権限」というセキュアな設計パターンを Node ネイティブに実現できる。

---

## 5. Nodemailer 9.0 リリース — TLS 検証をデフォルト有効化

**リンク:** https://github.com/nodemailer/nodemailer/releases  
**Changelog:** https://github.com/nodemailer/nodemailer/blob/master/CHANGELOG.md

### ① 目的・問題意識

Nodemailer で HTTPS リモートコンテンツ（添付ファイルの URL 取得、OAuth2 トークンエンドポイント、HTTP/HTTPS プロキシ CONNECT）へのリクエストが、以前は TLS 証明書の検証を行っていなかった。9.0 でセキュリティデフォルトを強化し、自己署名・期限切れ・ホスト名不一致の証明書へのリクエストをデフォルトで失敗させる。

### ② 具体的な技術詳細・コード

**破壊的変更（Breaking Change）:**
```javascript
// 8.x まで: 証明書検証なし（暗黙的に rejectUnauthorized: false）
const transporter = nodemailer.createTransport({
  host: 'smtp.example.com',
  port: 465,
  secure: true,
});
// 自己署名証明書のサーバーでも接続成功していた

// 9.0 から: TLS 証明書検証がデフォルト有効
// 自己署名・期限切れ・ホスト名不一致 → エラー
```

**移行方法（後方互換が必要な場合）:**
```javascript
// オプション1: transport レベルで検証を無効化（非推奨）
const transporter = nodemailer.createTransport({
  host: 'smtp.example.com',
  port: 465,
  secure: true,
  tls: {
    rejectUnauthorized: false, // 自己署名を許容（本番非推奨）
  },
});

// オプション2: 添付ファイル単位で無効化
const mailOptions = {
  attachments: [{
    href: 'https://internal-server/file.pdf',
    tls: { rejectUnauthorized: false }, // この添付のみ緩和
  }]
};
```

**その他のセキュリティ改善:**
```javascript
// SMTP コマンドインジェクション対策
// transport の name オプションに CRLF が含まれる場合サニタイズ

// OAuth2 / Ethereal への HTTPS リクエストでも厳格な TLS 検証を適用
// disableFileAccess / disableUrlAccess を raw message オプションにも適用
```

### ③ なぜ効果的か

「デフォルト安全（Secure by Default）」への移行は、設定ミスによるリモートコンテンツへの中間者攻撃リスクを構造的に排除する。既存環境への影響があるため Breaking Change だが、適切な証明書を使用しているプロダクション環境では変更不要。開発環境や内部サーバーには個別に `rejectUnauthorized: false` を設定するパターンを推奨。

---

## 6. VoidZero が Cloudflare に参加 — Vite エコシステムの新展開

**リンク:** https://blog.cloudflare.com/voidzero-joins-cloudflare/  
**VoidZero 発表:** https://voidzero.dev/posts/voidzero-cloudflare

### ① 目的・問題意識

Vite・Vitest・Rolldown（Rust 製バンドラー）・Oxc（Rust 製ツールチェーン）を擁する VoidZero が Cloudflare に買収（2026 年 6 月 4 日）。130 M/週のダウンロードを誇る Vite エコシステムを、Cloudflare の AI ネイティブ Web 戦略の中核に据える。

### ② 具体的な技術詳細

**買収対象ツール:**
- **Vite** — フロントエンドビルドツール（130M 週次ダウンロード）
- **Vitest** — Vite ベースのテストランナー
- **Rolldown** — Rust 製の高速バンドラー（Rollup 互換 API）
- **Oxc** — Rust 製 JavaScript/TypeScript ツールチェーン（パーサー、リンター、トランスパイラー）
- **Vite+** — Vite のプレミアム拡張

**オープンソースへの約束:**
- Vite / Vitest / Rolldown / Oxc / Vite+ はオープンソース・ベンダーニュートラル・コミュニティ主導を継続
- **Cloudflare が Vite エコシステムファンドに $1M を拠出**（Vite コアチームが管理）
- Evan You（Vue.js 作者）と VoidZero チーム全員が Cloudflare に移籍し、継続してリードする

**Rolldown の位置付け:**
```javascript
// Rolldown は Rollup 互換のAPIを持つ Rust 製バンドラー
// Vite 8 では Rolldown がデフォルトバンドラーになる見込み
// 既存の Vite プロジェクトは設定変更なしに移行できる設計

// vite.config.ts（Vite 8 以降の想定）
export default defineConfig({
  // Rolldown が自動的に使用される
  build: {
    rollupOptions: { /* 既存の Rollup オプションがそのまま動く */ }
  }
});
```

### ③ なぜ効果的か

Cloudflare の Infrastructure（CDN、Workers、R2 等）と Vite エコシステムの統合により、AI ネイティブ Web 開発のビルドパイプライン全体を Cloudflare プラットフォームで完結させる戦略。Rust 製ツール（Rolldown/Oxc）への投資加速が見込まれ、JavaScript ツールチェーンの大幅な高速化が期待される。$1M のエコシステムファンドにより、コミュニティメンテナーへの持続的な資金援助が行われる。

---

*Sources: [Node Weekly #629](https://nodeweekly.com/issues/629) / [nubjs.com](https://nubjs.com/) / [nodejs.org security releases](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases) / [TypeScript 7.0 RC](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/) / [Node.js 26.3.0](https://nodejs.org/en/blog/release/v26.3.0) / [Nodemailer releases](https://github.com/nodemailer/nodemailer/releases) / [VoidZero joins Cloudflare](https://blog.cloudflare.com/voidzero-joins-cloudflare/)*
