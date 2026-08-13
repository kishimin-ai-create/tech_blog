# Hono + Bun + MySQL/Drizzle でバックエンド基盤を構築する

## はじめに

**TypeScript をコンパイルなしで実行できる。** `tsc` ビルドステップや `ts-node` が不要で、`bun run src/index.ts` がそのまま動く。CI でのバックエンドビルドステップを削除できた（後述）。

## 対象読者

- TypeScript でバックエンドを書いているエンジニア
- Bun を実プロジェクトに導入したいエンジニア
- Drizzle ORM の導入を検討しているエンジニア

---

## 概要

`diary` モノレポのバックエンドを立ち上げるにあたり、以下のスタックを選定した。

| レイヤー | 採用技術 |
|---|---|
| ランタイム | Bun (latest) |
| HTTP フレームワーク | Hono 4 |
| データベースドライバ | mysql2 3.x |
| ORM / スキーマ管理 | drizzle-orm + drizzle-kit |
| バリデーション / OpenAPI | @hono/zod-openapi + zod + @hono/swagger-ui |
| リンター | ESLint flat config (typescript-eslint + 各種プラグイン) |
| フォーマッター | Prettier（リポジトリルートで一元管理） |
| 型チェック | @typescript/native-preview (tsgo) |

---

## Bun を選んだ理由

Node.js ベースの構成と比較したとき、Bun には実開発上で影響が大きい特徴がある。

**TypeScript をコンパイルなしで実行できる。** `tsc` ビルドステップや `ts-node` が不要で、`bun run src/index.ts` がそのまま動く。CI でのバックエンドビルドステップを削除できた（後述）。

**ホットリロードが組み込まれている。** `bun run --hot src/index.ts` で起動するだけで、ファイル変更時に自動リロードが走る。

**テストランナーが内蔵されている。** Vitest や Jest を追加インストールせず `bun test` が使える。グロブパターンで対象ファイルを絞れるため、テストサイズ規約（後述）との相性が良い。

---

## Hono 4 のセットアップ

`package.json` の依存関係は最小限に保った。

```json
{
  "dependencies": {
    "drizzle-orm": "^0.45.2",
    "hono": "^4.12.23",
    "mysql2": "^3.22.4"
  }
}
```

Hono 自体は `dependencies` に置き、OpenAPI 関連 (`@hono/zod-openapi`, `@hono/swagger-ui`, `@hono/standard-validator`) と ORM ツール (`drizzle-kit`, `drizzle-zod`) は `devDependencies` に分類した。

### OpenAPI / バリデーション

`@hono/zod-openapi` を使うと、Zod スキーマからルート定義と OpenAPI spec を同時に生成できる。フロントエンドで Orval によるクライアントコード自動生成を行う前提があるため、バックエンドが OpenAPI spec を常に最新に保てる構成は必須だった。

---

## MySQL + Drizzle ORM

### 環境変数

`.env.example` に必要な変数を明示してある。

```ini
DB_HOST=localhost
DB_PORT=3306
DB_NAME=diary_db
DB_USER=root
DB_PASSWORD=
DATABASE_URL=mysql://root:@localhost:3306/diary_db
```

`.env` は `.gitignore` で除外し、`.env.example` をコピーして使う。

### drizzle-kit スクリプト

```json
{
  "db:generate": "drizzle-kit generate",
  "db:migrate": "drizzle-kit migrate",
  "db:studio": "drizzle-kit studio"
}
```

スキーマ変更 → `db:generate` でマイグレーションファイル生成 → `db:migrate` で適用、という流れを統一している。開発中は `db:studio` で Drizzle Studio（ブラウザ GUI）を起動してデータを確認できる。

---

## ESLint flat config

`eslint.config.mts` に全設定を集約した。主要プラグインと目的を整理すると以下のとおり。

| プラグイン | 目的 |
|---|---|
| `typescript-eslint` (recommended + recommendedTypeChecked) | TypeScript の型安全ルール |
| `eslint-plugin-security` | バックエンド特有のセキュリティリスク検出 |
| `eslint-plugin-drizzle` | Drizzle ORM 固有の危険パターン防止 |
| `eslint-plugin-simple-import-sort` | import 順序の自動整列 |
| `eslint-plugin-unused-imports` | 未使用 import の検出・自動削除 |
| `eslint-plugin-jsdoc` | public 関数の JSDoc 必須化 |
| `@stylistic/eslint-plugin` | セミコロン等のスタイル統一 |
| `eslint-config-prettier` | ESLint と Prettier のルール競合を解消（最後に配置） |

### Drizzle 固有ルール

```ts
"drizzle/enforce-delete-with-where": "error",
"drizzle/enforce-update-with-where": "error",
```

WHERE 句なしの UPDATE / DELETE を静的解析でエラーにする。ORM を使っていても誤って全行更新・全行削除するミスを防ぐためのルールで、バックエンド API では特に重要な安全網になる。

### TypeScript 型チェックルール（抜粋）

```ts
"@typescript-eslint/switch-exhaustiveness-check": "warn",
"@typescript-eslint/consistent-type-imports": ["error", { prefer: "type-imports" }],
"@typescript-eslint/prefer-nullish-coalescing": "warn",
"@typescript-eslint/no-unnecessary-condition": "warn",
```

型情報を使った解析 (`recommendedTypeChecked`) を有効にすることで、`switch` の網羅性チェックや不要な条件分岐の検出が静的に行える。

---

## 型チェック：@typescript/native-preview (tsgo)

```json
{
  "typecheck": "bun run --bun tsgo --noEmit"
}
```

通常の `tsc` の代わりに `@typescript/native-preview`（`tsgo`）を使っている。これは TypeScript チームが開発中のネイティブ実装のプレビューパッケージで、`tsc` より高速に型チェックを完了できる。

**注意点：** このパッケージはプレビュー版であり、破壊的変更が入る可能性がある。プロダクション用途では安定性の評価が必要。

---

## テストスクリプト

```json
{
  "test:small":    "bun test small.test",
  "test:medium":   "bun test medium.test",
  "test:large":    "bun test large.test",
  "test:coverage": "bun test --coverage"
}
```

`bun test` はグロブ引数でファイル名にマッチするテストだけを実行できる。`small.test`、`medium.test`、`large.test` という文字列がファイル名に含まれるテストだけを対象にする設計で、CI の各ステージが適切なテストだけを実行できる。

テストサイズ規約の詳細と CI パイプライン全体の設計については別記事で扱う。

---

## スクリプト全体像

```json
{
  "dev":          "bun run --hot src/index.ts",
  "typecheck":    "bun run --bun tsgo --noEmit",
  "lint":         "eslint .",
  "lint:fix":     "eslint --fix .",
  "format":       "prettier --write .",
  "format:check": "prettier --check .",
  "check":        "eslint . && prettier --check .",
  "check:fix":    "eslint --fix . && prettier --write ."
}
```

`check` / `check:fix` はリントとフォーマット確認を一括実行するショートカット。CI では `lint` と `typecheck` を独立したステップとして実行するため、コマンドは細かく分離してある。

---

## まとめ

- **Bun を選ぶと TS→JS コンパイルステップが消える**。CI が単純化され、ローカル開発の起動も速い
- **Hono + @hono/zod-openapi** の組み合わせでルート定義・バリデーション・OpenAPI spec を1箇所で管理できる
- **Drizzle の ESLint プラグイン**は WHERE 句忘れを静的に防ぐ実用的な安全策
- **eslint-config-prettier を最後に配置する**ことで ESLint と Prettier の競合をゼロにできる
- **@typescript/native-preview (tsgo)** は高速だがプレビュー段階のため、アップグレード時の破壊的変更に注意が必要
