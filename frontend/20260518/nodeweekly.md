# Node Weekly Issue 624 (May 14, 2026) まとめ

> 出典: https://nodeweekly.com/issues/624

---

## 1. NodeBook: An Advanced Guide to Node.js Internals — Volume I 完成

**リンク:** https://www.thenodebook.com/

### ① 目的・問題意識

Node.js の「なんとなく動く」から「なぜ動くか」を理解したいエンジニアに向けた、内部実装レベルの詳細ガイド。公式ドキュメントが省略しているイベントループの真の挙動・V8 の最適化機構・Buffer の低レベル実装・async/await の脱糖変換などを徹底解説する。Volume I が全8章で完成し、Volume II（Network I/O）が進行中。

### ② 技術詳細

**カバー範囲:**
- **イベントループ内部:** Libuv が実装する6フェーズ（timers → pending callbacks → idle → poll → check → close）の詳細。各フェーズ間でマイクロタスクキューが完全にドレインされる仕組み。
- **V8 の動作:** TurboFan オプティマイザーによる JIT コンパイル、Hidden Class の遷移、Polymorphic Inline Cache ミスのデバッグ方法。
- **Buffer 割り当て:** V8 ヒープ外のメモリ確保、concurrent traffic 下でのヒープ枯渇リスクと `highWaterMark` を用いたバックプレッシャー制御。
- **Streams:** メモリリークしないストリーム実装パターン。
- **モジュール解決:** Node.js がモジュールを検索するアルゴリズムの詳細。
- **async/await の実体:**

```javascript
// async/await が実際に行うこと（概念的な脱糖）
async function fetchData() {
  const result = await fetch('/api/data');
  return result.json();
}

// 内部的には以下に近い処理が走る:
// 1. Promise オブジェクトを V8 ヒープに生成
// 2. 内部の resolve/reject 関数を割り当て
// 3. executor をラップ
// 4. settled 時にマイクロタスクキューへのスケジューリングを実施
// TurboFan は Promise の解決パスをインライン化し、
// 中間オブジェクトの生成を最小化するよう最適化
```

**イベントループとマイクロタスクの優先順位:**

```
[各イベントループフェーズ終了後]
  → process.nextTick キューを完全ドレイン
  → Promise マイクロタスクキューを完全ドレイン
  → 次のフェーズへ

// 例: nextTick は Promise より先に実行される
process.nextTick(() => console.log('1: nextTick'));
Promise.resolve().then(() => console.log('2: Promise'));
console.log('0: sync');
// 出力: 0: sync → 1: nextTick → 2: Promise
```

**Libuv が OS の非同期 I/O を抽象化する構造:**

| OS      | 非同期 I/O 機構  |
|---------|----------------|
| Linux   | epoll          |
| macOS   | kqueue         |
| Windows | IOCP           |

Libuv はこれら全てに対し単一の C API を提供し、Node.js コードのクロスプラットフォーム性を担保している。

**プロセスライフサイクル:**

```
node executable 起動
  → CLI フラグ解析
  → V8 初期化・Isolate/Context 生成
  → libuv 初期化
  → ネイティブモジュール登録
  → 内部ブートストラップ JavaScript 実行
  → エントリーモジュール読み込み
  → 参照カウントのある handle/request/timer/socket/worker が
    存在する間、イベントループを継続
```

### ③ なぜ効果的か

「仕組みを知る」ことで、パフォーマンスボトルネックの根本原因を特定できる。例えば、イベントループのどのフェーズでブロッキングが起きているかを把握できれば、Worker Threads や非同期化の投入箇所を正確に判断できる。V8 の最適化の仕組みを理解することで、意図せず JIT 最適化を無効化するコードパターン（例：polymorphic 関数への型混在）を回避できる。

---

## 2. Node.js 26.1.0 (Current) リリース

**リンク:** https://nodejs.org/en/blog/release/v26.1.0

### ① 目的・問題意識

Node.js 26.0.0（2026年5月5日リリース）に続く Current リリース。実験的 FFI モジュール・UUIDv7 生成・ファイルシステム操作のキャンセル信号など、開発者の利便性と相互運用性を向上させる機能群を追加。

### ② 技術詳細

**[主要機能 1] 実験的 FFI モジュール (`node:ffi`)**

ネイティブの動的ライブラリ（.dll / .so / .dylib）を JavaScript から直接ロード・呼び出す機能。C 呼び出し規約に対応した任意の言語のバイナリと連携可能になる。

```bash
# 使用には --experimental-ffi フラグが必要
node --experimental-ffi app.js

# Permission Model 使用時は --allow-ffi も必要
node --experimental-ffi --allow-ffi app.js
```

> **安全上の注意:** この API は本質的に unsafe。不正なポインタ・シグネチャの誤り・解放済みメモリへのアクセスはプロセスクラッシュまたはメモリ破壊を引き起こす。（Contributor: Paolo Insogna, PR #62072）

---

**[主要機能 2] `crypto.randomUUIDv7()`**

時刻ソート可能な UUID v7 を生成する組み込み関数。サードパーティパッケージ不要で単調増加する一意識別子を生成できる。

```javascript
import { randomUUIDv7 } from 'node:crypto';

const id = randomUUIDv7();
// 例: "01952c88-88bc-7000-b123-456789abcdef"
// 先頭 48bit がミリ秒タイムスタンプ → 自然に時系列ソート可能

// UUID v4 との比較:
import { randomUUID } from 'node:crypto';
const v4 = randomUUID(); // 完全ランダム、ソート不可
const v7 = randomUUIDv7(); // 時刻順、データベースのインデックス効率が向上
```

---

**[主要機能 3] ファイルシステム操作へのキャンセル信号**

`fs.stat()` 等のファイルシステム操作に `AbortSignal` によるキャンセルを追加。

```javascript
import { stat } from 'node:fs/promises';

const controller = new AbortController();
const { signal } = controller;

// タイムアウト付きのファイル stat
const statPromise = stat('/path/to/large/file', { signal });

// 500ms 後にキャンセル
setTimeout(() => controller.abort(), 500);

try {
  const stats = await statPromise;
  console.log(stats.size);
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('stat operation was cancelled');
  }
}
```

---

**[依存関係の更新]**
- V8: 14.6.202.34 へパッチ適用
- npm: 11.13.0
- undici: 8.1.0
- sqlite: 3.53.0
- timezone: 2026b

### ③ なぜ効果的か

- **FFI:** C/C++ バインディングのために node-gyp・Python・コンパイル環境を用意する必要がなくなる可能性がある。ただし実験的フェーズ。
- **UUIDv7:** データベースの B-tree インデックスにおいて、時刻順 UUID はランダム UUID より書き込み効率が大幅に向上する。MongoDB・PostgreSQL の主キーとして有効。
- **キャンセル信号:** HTTP リクエストキャンセル時に関連するファイル I/O も連動して停止でき、リソースリークを防止できる。

---

## 3. TypeScript 7.0 Beta — Go ネイティブポートで 10x 高速化

**リンク:** https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-beta/

### ① 目的・問題意識

大規模 TypeScript プロジェクトでのコンパイル時間が深刻なボトルネックになっている（VS Code の 1.5M 行コードベースで ~90 秒）。Go 言語でネイティブ実装することで、並列処理と JIT 不要の実行速度によって約 10 倍の高速化を実現する（プロジェクトコードネーム: **Project Corsa**）。

### ② 技術詳細

**パフォーマンス比較:**

| コードベース     | TypeScript 6.0 | TypeScript 7.0 (tsgo) | 速度改善 |
|----------------|---------------|----------------------|---------|
| VS Code (1.5M行)| ~89 秒         | ~7.5 秒               | 10.4x   |
| Sentry          | ~133 秒        | ~16 秒               | 8.3x    |
| 型チェックのみ   | -              | -                    | ~30x    |

**インストールと使用方法:**

```bash
# グローバルインストール
npm install -g @typescript/native-preview

# または beta 版
npm install -g @typescript/native-preview@beta

# tsgo コマンドとして実行 (tsc と同様のインターフェース)
tsgo --noEmit
tsgo --project tsconfig.json

# 既存の tsc と並列実行して比較可能
tsc --noEmit          # 従来の JS ベースコンパイラ
tsgo --noEmit         # 新しい Go ベースコンパイラ
```

**VS Code での使用:**

```json
// .vscode/settings.json
{
  "typescript.experimental.useTsGo": true
}
// TypeScript Native Preview 拡張機能を別途インストール
```

**設計思想:**
- 既存の TypeScript コードベースを Go に**メソッド論的に移植**（スクラッチからの書き直しではない）
- 型チェックロジックは TypeScript 6.0 と**構造的に同一**
- 共有メモリ並列処理によるマルチスレッド型チェック

**互換性上の注意点:**

```
TypeScript 7.0 は Strada API（旧コンパイラ API）をサポートしない
→ ESLint TypeScript プラグイン・フォーマッタ等が依存するため
   移行期は以下の並列運用を推奨:

tsgo      → 高速な型チェック用
tsc (v6)  → ツールチェーン（linter 等）との互換性用

TypeScript 7.1 でツールチェーン向け安定 API が提供予定
```

### ③ なぜ効果的か

Go は GC オーバーヘッドが少なく、ゴルーチンによる共有メモリ並列処理が可能。JavaScript（現行 tsc）は V8 の JIT 最適化に依存しており、並列処理のための SharedArrayBuffer の使用も制約が多い。ネイティブコードで直接実行することで、特に大規模モノレポや CI/CD パイプラインでのフィードバックループが劇的に短縮される。

---

## 4. TypeScript 6.0 リリース — JavaScript ベースの最後のバージョン

**リンク:** https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/

### ① 目的・問題意識

TypeScript 7.0（Go ネイティブ実装）へのスムーズな移行路を提供するブリッジリリース。10年分の技術的負債（非推奨パターン、Legacy 構文、互換性のためのデフォルト設定）を清算し、7.0 移行前に開発者がコードベースを整備できるようにする。

### ② 技術詳細

**主要な破壊的変更 / 非推奨化:**

```typescript
// ① target: "es5" が非推奨（最低ターゲットは ES2015）
// tsconfig.json
{
  "compilerOptions": {
    "target": "es5"  // ❌ TypeScript 6.0 で非推奨エラー
    // "target": "es2015"  // ✅ 推奨
  }
}

// ② --downlevelIteration フラグが非推奨
// ES5 ターゲット廃止に伴い不要になった

// ③ import assertions 構文が非推奨（import attributes に移行）
import data from './data.json' assert { type: 'json' }; // ❌ 旧構文
import data from './data.json' with { type: 'json' };   // ✅ 新構文

// ④ namespace キーワードの代わりに module を使う書き方が廃止
// （module blocks という ECMAScript 提案と構文衝突するため）
module MyModule {       // ❌ 旧構文
  export function foo() {}
}
namespace MyNamespace { // ✅ namespace を使うこと
  export function foo() {}
}
```

**新機能:**

```typescript
// ① Temporal API のサポート（Node.js 26 で有効化済み）
const now = Temporal.Now.instant();
const date = Temporal.PlainDate.from('2026-05-14');
const duration = Temporal.Duration.from({ days: 30 });
const future = date.add(duration);
// → Temporal.PlainDate { year: 2026, month: 6, day: 13 }

// ② ES2025 lib の追加
// tsconfig.json
{
  "compilerOptions": {
    "lib": ["ES2025"],  // Set.union(), Set.intersection() 等が型定義に含まれる
    "target": "ES2025"
  }
}

// ③ --stableTypeOrdering フラグ（6.0→7.0 移行支援）
// 型の順序を安定化させ、Go 版コンパイラとの差異を最小化
```

**デフォルト値の変更:**

| オプション  | TS 5.x デフォルト | TS 6.0 デフォルト |
|------------|-----------------|-----------------|
| `strict`   | false           | true            |
| `module`   | CommonJS        | Node16           |
| `target`   | ES3             | ES2015          |

### ③ なぜ効果的か

「後方互換性のために残していた Legacy モード」を廃止することで、Go ネイティブコンパイラが省略できる変換処理が減少し、7.0 の高速化がより実現しやすくなる。`--stableTypeOrdering` フラグにより、6.0 と 7.0 で型チェック結果が一致することを事前に確認でき、移行リスクを低減できる。

---

## 5. A Gentle Intro to npm Workspaces — 図解付き入門

**リンク:** https://wasp.sh/blog/2026/03/25/gentle-intro-npm-workspaces

### ① 目的・問題意識

モノレポ（複数パッケージを1リポジトリで管理する構成）において、npm の依存関係解決アルゴリズムや hoisting の仕組みを正しく理解せずにハマる開発者が多い。本記事は図解付きで「npm workspaces が何をやっているか」を内部動作レベルで解説する。

### ② 技術詳細

**基本セットアップ:**

```
monorepo/
├── package.json          ← ルート（private: true が必須）
├── node_modules/         ← 共有依存関係が hoisting される場所
└── packages/
    ├── app/
    │   └── package.json
    ├── ui/
    │   └── package.json
    └── utils/
        └── package.json
```

```json
// ルート package.json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "packages/*"
  ]
}
```

```json
// packages/app/package.json — 同リポジトリ内の ui パッケージを参照
{
  "name": "@myapp/app",
  "version": "1.0.0",
  "dependencies": {
    "@myapp/ui": "workspace:*",   // ローカルワークスペースを参照
    "react": "^19.0.0"
  }
}
```

**依存関係の Hoisting（巻き上げ）:**

npm は共有依存関係を最上位 `node_modules/` に配置し、各パッケージが個別にコピーを持たないようにする。

```
# 3つのパッケージが react@19 に依存している場合:
node_modules/
├── react/           ← ルートに1つだけ配置（hoisted）
├── @myapp/
│   ├── app/        ← symlink → packages/app
│   ├── ui/         ← symlink → packages/ui
│   └── utils/      ← symlink → packages/utils

# react がない場合は packages/app/node_modules/ に個別に配置される
```

**バージョン競合時の解決:**

```
# left-pad@1.0.0 と right-pad@1.0.0 が
# 異なるバージョンの core-pad に依存している場合:

node_modules/
├── left-pad/
├── right-pad/
├── core-pad/          ← 互換バージョンがここに hoisted
└── right-pad/
    └── node_modules/
        └── core-pad/  ← 互換しないバージョンはここに分離
```

**よくある落とし穴:**

```bash
# ❌ ワークスペース内で npm install を実行するとシンボリックリンクが壊れる
cd packages/app && npm install

# ✅ 常にルートで実行すること
npm install                              # 全ワークスペースに適用
npm install lodash -w packages/app       # 特定ワークスペースのみ
npm run build --workspaces               # 全ワークスペースでスクリプト実行
npm run test -w packages/app             # 特定ワークスペースのみ
```

**クロスパッケージインポート:**

```typescript
// packages/app/src/index.ts
// @myapp/ui は workspace:* で依存宣言されているため、
// ルートの node_modules/@myapp/ui symlink 経由で解決される
import { Button } from '@myapp/ui';
import { formatDate } from '@myapp/utils';
```

### ③ なぜ効果的か

npm の既存のモジュール解決アルゴリズム（node_modules 上位階層への順次探索）をそのまま活用することで、ランタイムの変更が不要。シンボリックリンクにより、ローカルパッケージを `npm publish` せずに相互参照でき、開発中の変更が即座に反映される。Hoisting により、`react` のような大きな依存関係が複数コピーになることを防ぎ、ディスク使用量と `require` の解決時間を削減できる。

---

## 6. Knip v6 リリース — oxc-parser による高速化

**リンク:** https://knip.dev/blog/knip-v6

### ① 目的・問題意識

JavaScript/TypeScript プロジェクトには、時間の経過とともに使われなくなったファイル・エクスポート・依存関係が蓄積する。Knip はこれらを静的解析で検出するツールだが、従来の TypeScript バックエンドは IDE 向けの重い API を使用しており処理が遅かった。v6 では oxc-parser に切り替えて 2〜4 倍の高速化を実現。

### ② 技術詳細

**インストールと基本使用:**

```bash
npm install -D knip typescript @types/node

# package.json にスクリプト追加
# { "scripts": { "knip": "knip" } }

npm run knip
```

**設定ファイル例 (`knip.json`):**

```json
{
  "$schema": "https://unpkg.com/knip/schema.json",
  "entry": ["src/index.ts", "src/cli.ts"],
  "project": ["src/**/*.ts"],
  "ignore": ["**/__mocks__/**"],
  "ignoreDependencies": ["some-peer-dep"],
  "ignoreExportsUsedInFile": true
}
```

**検出できる問題の例:**

```typescript
// src/utils.ts
export function usedFn() { return 'used'; }
export function unusedFn() { return 'unused'; } // ← Knip が検出

// src/types.ts
export type UsedType = string;
export type UnusedType = number; // ← Knip が検出
```

```bash
# 実行結果の例:
Unused files (1)
  src/legacy-helper.ts

Unused exports (2)
  src/utils.ts: unusedFn
  src/types.ts: UnusedType

Unused dependencies (1)
  lodash (listed in package.json but never imported)
```

**v6 の技術的変更点:**

- **TypeScript バックエンド → oxc-parser に置換**  
  静的解析専用に oxc-parser（Rust 実装）と oxc-resolver を採用。IDE 向けプログラム配線（型チェッカーのオーバーヘッド）が不要なため、各ファイルを1度だけ解析すれば足りる。
  
- **パフォーマンス改善:** 2〜4 倍高速化

- **プラグインの静的解析化:**  
  ESLint（flat config）・tsdown・tsup プラグインがコンフィグファイルを直接静的解析するよう改良。

- **動作環境:** Node.js v20.19.0 以上が必要（v18 サポート終了）

**CI/CD への統合:**

```yaml
# .github/workflows/knip.yml
- name: Run Knip
  run: npx knip --no-exit-code  # 問題があっても CI を止めない場合
  # または
  run: npx knip                 # 未使用コードがあれば exit code 1 で失敗
```

### ③ なぜ効果的か

oxc-parser は Rust で書かれており、TypeScript の JS 実装より桁違いに高速にファイルをパース・転送できる。さらに、IDE 用の完全な型プログラムを構築せずに純粋に AST ベースの静的解析のみを行うことで、不要な計算を排除している。プロジェクトが大きくなるほど効果が顕著で、不要なコードを定期的に除去することでバンドルサイズ削減・メンテナンスコスト低下・テストの信頼性向上につながる。

---

## 7. Writing Node.js Addons with .NET Native AOT — C# で Node.js アドオンを書く

**リンク:** https://devblogs.microsoft.com/dotnet/writing-nodejs-addons-with-dotnet-native-aot/

### ① 目的・問題意識

Node.js のネイティブアドオンは従来 C++ と `node-gyp` で書く必要があり、Python・ビルドチェーン・プラットフォーム固有の設定が必要で複雑だった。Microsoft の C# Dev Kit チームが、アドオンを **C# + .NET 10 Native AOT** で書き直し、C++ と同等のパフォーマンスを C# で実現できることを実証した。

### ② 技術詳細

**仕組みの概要:**

```
Node.js のネイティブアドオン = N-API（stable ABI）を実装した共有ライブラリ
→ C 呼び出し規約さえ使えれば任意の言語で実装可能
→ .NET 10 の [UnmanagedCallersOnly] がこれを実現
```

**C# での実装例:**

```csharp
// MyAddon.cs
using System;
using System.Runtime.InteropServices;

// プロジェクトファイル設定（MyAddon.csproj）:
// <PublishAot>true</PublishAot>
// <AllowUnsafeBlocks>true</AllowUnsafeBlocks>

public unsafe class MyAddon
{
    // Node.js が最初に呼び出すエントリーポイント
    [UnmanagedCallersOnly(
        EntryPoint = "napi_register_module_v1",
        CallConvs = new[] { typeof(System.Runtime.CompilerServices.CallConvCdecl) }
    )]
    public static napi_value Init(napi_env env, napi_value exports)
    {
        // "hello"u8 は UTF-8 バイト文字列リテラル（N-API が要求する形式）
        fixed (byte* name = "sayHello"u8)
        {
            napi_create_function(env, name, /* auto-length */ -1,
                &SayHello, null, out napi_value fn);
            napi_set_named_property(env, exports, name, fn);
        }
        return exports;
    }

    // &SayHello は UnmanagedCallersOnly により関数ポインタとして取得可能
    // （ジェネリクスや async は使用不可）
    [UnmanagedCallersOnly(CallConvs = new[] {
        typeof(System.Runtime.CompilerServices.CallConvCdecl) })]
    private static napi_value SayHello(napi_env env, napi_callback_info info)
    {
        napi_create_string_utf8(env, "Hello from C#!"u8, out napi_value result);
        return result;
    }
}
```

**ビルドと使用:**

```bash
# AOT ネイティブライブラリとして公開
dotnet publish -r linux-x64 -o ./dist

# Node.js から使用
```

```javascript
// JavaScript 側での使用
const addon = require('./dist/MyAddon.node');

console.log(addon.sayHello()); // → "Hello from C#!"
```

**高レベルフレームワークの使用（node-api-dotnet）:**

```csharp
// node-api-dotnet を使うとボイラープレートを大幅削減
using Microsoft.JavaScript.NodeApi;

[JSExport]
public class Calculator
{
    public static int Add(int a, int b) => a + b;
    public static string Greet(string name) => $"Hello, {name}!";
}
```

```javascript
// JavaScript から透過的に使用
const { Calculator } = require('./Calculator.node');
console.log(Calculator.Add(1, 2));    // → 3
console.log(Calculator.Greet('Node')); // → "Hello, Node!"
```

**制約事項:**

| 項目 | 内容 |
|------|------|
| バイナリサイズ | 少なくとも 4〜10 MB（プラットフォーム依存） |
| 呼び出し可能なコード | ネイティブコードのみ（managed .NET アセンブリ不可） |
| 使用不可の機能 | 動的ロード・リフレクション・ランタイムコード生成 |
| クロスプラットフォーム | Windows / Linux / macOS で動作 |

### ③ なぜ効果的か

- **Python 不要:** `node-gyp` のビルドチェーンを排除でき、CI/CD のセットアップが単純化される。
- **C++ と同等のパフォーマンス:** Native AOT はネイティブコードを生成するため、マネージドランタイムのオーバーヘッドがなく、文字列マーシャリングやレジストリアクセスなどの実測パフォーマンスは C++ 版と同等。
- **C# の開発体験:** 型安全性・モダンな言語機能・豊富なライブラリエコシステムを活用しつつ、Node.js アドオンを書ける。
- **安定 ABI:** N-API の stable ABI により、Node.js のバージョンアップでバイナリの再コンパイルが不要。

---

*Node Weekly Issue 624 (May 14, 2026) — まとめ終了*
