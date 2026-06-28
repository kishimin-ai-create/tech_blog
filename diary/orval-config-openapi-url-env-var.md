# orval.config.ts のハードコード URL を環境変数へ移行した

## 対象読者

- Orval を使ってフロントエンドの API クライアントを自動生成しているエンジニア
- 「localhost しか叩けない」生成スクリプトを複数環境に対応させたい方
- リポジトリの `no-hardcoded-urls` ルールに準拠した設定ファイルの書き方を知りたい方

---

## 問題の背景

フロントエンドでは [Orval](https://orval.dev/) を使い、バックエンドの OpenAPI スキーマから TypeScript 型・React Query フック・Zod バリデーターを自動生成している。生成は次のコマンドで実行する。

```bash
bun run api:generate   # 内部的に orval を呼ぶ
```

このコマンドが参照する設定ファイル `frontend/orval.config.ts` には、2 つの入力ターゲット（`diary` と `diaryZod`）があり、どちらも OpenAPI の取得先 URL を直書きしていた。

```ts
// 修正前
export default defineConfig({
  diary: {
    input: {
      target: "http://localhost:3000/openapi/v1.json", // ← ハードコード
    },
    // ...
  },
  diaryZod: {
    input: {
      target: "http://localhost:3000/openapi/v1.json", // ← 同じ文字列が重複
    },
    // ...
  },
});
```

---

## 原因・制約

この書き方には 2 つの問題があった。

### 1. リポジトリルール違反

本リポジトリには **`no-hardcoded-urls`** というルールがあり、ソースファイル内に `http://` を含む生文字列リテラルを直接書くことを禁止している。URL は必ず環境変数経由で渡す必要がある。

### 2. 環境をまたいだ利用ができない

`localhost:3000` を直書きしていると、ステージングや本番の OpenAPI スペックに対して生成を走らせたい場合でもソースコードを書き換えるしかなかった。これは CI / CD パイプラインや複数環境への対応を妨げる。

---

## 解決策

ファイル冒頭に `OPENAPI_URL` 定数を 1 つ宣言し、両方の入力ターゲットからその定数を参照するように変更した。

```ts
// 修正後: frontend/orval.config.ts
import { defineConfig } from "orval";

const OPENAPI_URL =
  process.env.OPENAPI_URL ?? "http://localhost:3000/openapi/v1.json";

export default defineConfig({
  diary: {
    input: {
      target: OPENAPI_URL,
    },
    // ...
  },
  diaryZod: {
    input: {
      target: OPENAPI_URL,
    },
    // ...
  },
});
```

---

## 実装のポイント

### `??`（Nullish Coalescing）でフォールバックを設定

`process.env.OPENAPI_URL ?? "http://localhost:3000/openapi/v1.json"` という書き方は、環境変数が未設定（`undefined`）の場合にのみフォールバック値を使う。これにより、ローカル開発では何も設定しなくても従来通り動作し、CI や別環境では環境変数を注入するだけで向き先を変えられる。

```bash
# ステージング向けに生成する場合の例
OPENAPI_URL=https://staging.example.com/openapi/v1.json bun run api:generate
```

### 重複の除去

修正前は同一の URL 文字列が `diary` と `diaryZod` の 2 箇所に分散していた。定数を 1 か所にまとめることで、向き先を変えるときの変更点が 1 か所に絞られ、片方だけ書き換え忘れるというミスがなくなる。

### 型チェックの確認

変更後に `bun run typecheck`（= `tsc --noEmit`）を実行し、exit code 0 で通過することを確認済み。

---

## 注意点

- `orval.config.ts` は Node.js / Bun のスクリプトとして実行されるため、`process.env` は実行時に評価される。`.env` ファイルを利用している場合は、`dotenv` などで事前に読み込む必要がある点に注意。
- フォールバック値として残している `http://localhost:3000/openapi/v1.json` はローカル開発専用の値であり、`no-hardcoded-urls` ルールの「フォールバックはソースに置いてよい」例外扱いとして機能する（環境変数が最優先される構造になっているため）。

---

## まとめ

| 項目 | 修正前 | 修正後 |
|------|--------|--------|
| URL の管理場所 | 設定ファイル内に直書き（2 か所） | 環境変数 `OPENAPI_URL`（定数 1 か所） |
| 環境切り替え | ソースを書き換えるしかない | 環境変数を差し替えるだけ |
| ルール準拠 | `no-hardcoded-urls` 違反 | 準拠 |
| TypeScript 型チェック | — | `tsc --noEmit` 通過 |

変更量は数行だが、「ローカルにしか向けられないスクリプト」が「環境変数 1 つで向き先を変えられるスクリプト」に変わり、CI パイプラインや複数環境での運用に耐えられるようになった。小さな修正でも、ルール違反を早めに直しておくことで後の運用コストを下げられる。
