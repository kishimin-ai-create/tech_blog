# テストの `import "crypto"` を `import "node:crypto"` に修正する

## はじめに

環境では、`node_modules/crypto` が解決先として優先されてしまうリスクがある。

## 対象読者

- Node.js / Bun でバックエンドを書いているエンジニア
- テストコードの import 書き方に迷ったことがある人
- モノレポ環境でパッケージ衝突を経験したことがある人

---

## 背景

`backend/src/services/auth.service.small.test.ts` に次の import があった。

```ts
import { scryptSync, randomBytes } from "crypto";
```

動作上は問題なく見えるが、このプロジェクトのプロダクションコードである
`backend/src/models/user.ts` はすでに `node:` プレフィックスを使っていた。

```ts
// backend/src/models/user.ts（変更前から）
import { randomBytes, scryptSync, timingSafeEqual } from "node:crypto";
```

テストコードだけが裸の識別子 `"crypto"` を使っており、コードベース内で表記が揺れていた。

---

## 原因：裸の識別子は npm パッケージにシャドウされる可能性がある

Node.js は `"crypto"` という裸の識別子を受け取ると、`node_modules/` を検索してから
組み込みモジュールへフォールバックする（ランタイムやバンドラーの実装による）。

実際に `crypto` という名前の npm パッケージは存在しており、もともとブラウザ向けの
ポリフィルとして公開されている。モノレポ環境やフロントエンドのビルドツールが同居している
環境では、`node_modules/crypto` が解決先として優先されてしまうリスクがある。

一方、`node:` プレフィックスを付けた識別子は **常に Node.js 組み込みモジュール** を
参照することが言語仕様として保証されており、`node_modules/` は一切検索されない。

---

## 解決策：`node:` プレフィックスを付ける

```diff
-import { scryptSync, randomBytes } from "crypto";
+import { randomBytes, scryptSync } from "node:crypto";
```

変更点は 2 つ。

| 変更 | 内容 |
|---|---|
| 識別子 | `"crypto"` → `"node:crypto"` |
| 並び順 | アルファベット順（`randomBytes` → `scryptSync`）に整列 |

これにより、テストコードとプロダクションコードの import スタイルが統一された。

---

## 実装詳細：`node:` プレフィックスが使えるバージョン

`node:` プレフィックスは Node.js **v14.18.0 / v16.0.0** から利用できる。
このプロジェクトは Bun ランタイムを使用しており、Bun は `node:` プレフィックスを
フルサポートしているため、互換性の問題はない。

```ts
// 修正後（テストファイル）
import { randomBytes, scryptSync } from "node:crypto";

// プロダクションコード（変更なし・参考）
import { randomBytes, scryptSync, timingSafeEqual } from "node:crypto";
```

テスト内では `randomBytes` と `scryptSync` を使ってテスト用パスワードハッシュを
生成している。これは `createTestPasswordHash()` ヘルパー関数で使われており、
プロダクション実装と同じアルゴリズムを再現してログイン系テストを成立させている。

---

## 注意点

**ESLint で自動強制する方法**

`eslint-plugin-unicorn` の `unicorn/prefer-node-protocol` ルールを有効にすると、
裸の識別子による組み込みモジュール import を自動検出できる。

```js
// eslint.config.js
{
  rules: {
    "unicorn/prefer-node-protocol": "error",
  },
}
```

このルールを導入しておけば、今後同様の表記揺れを CI で防止できる。

---

## まとめ

| 項目 | 内容 |
|---|---|
| 変更ファイル | `backend/src/services/auth.service.small.test.ts` |
| 変更内容 | `"crypto"` → `"node:crypto"` |
| 理由 | npm パッケージによるシャドウイングを防ぐ・プロダクションコードとの一貫性確保 |
| テスト結果 | 82/82 パス |
| TypeScript | エラー 0 件 |

Node.js 組み込みモジュールを import するときは `node:` プレフィックスを付けるのが
現代の推奨スタイルであり、コードベース全体で統一することでシャドウイングリスクを排除できる。
