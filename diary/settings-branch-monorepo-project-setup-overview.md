# diary モノレポ全体設計：settings ブランチで整備したプロジェクト基盤

## はじめに

Node.js + npm/yarn の代わりに Bun を選んだ最大の理由は、**TypeScript をコンパイルなしで実行できる**ことだ。バックエンドでは `bun run --hot src/index.ts` で直接起動でき、CI にビルドステップが不要になる。

## 対象読者

- TypeScript モノレポをゼロから立ち上げたいエンジニア
- バックエンドとフロントエンドで統一した品質ルールを整備したい人
- Bun をモノレポのパッケージマネージャー兼ランタイムとして使いたい人
- CI の実行戦略（push / PR / ナイトリーの使い分け）を検討しているエンジニア

---

## 概要

`diary` リポジトリの `settings` ブランチでは、アプリケーションコードはまだ存在しない。このブランチが整備したのは **「コードを正しく書き続けるための土台」** だ。

具体的には以下の4つの柱で構成される。

1. **バックエンドのスケルトン**：Hono + Bun + MySQL (drizzle-orm)
2. **フロントエンドのスケルトン**：Next.js 16 + React 19 + TanStack Query + Orval
3. **GitHub Actions による CI/CD パイプライン**：5ワークフロー
4. **両ワークスペース横断の品質ベースライン**：ESLint / Prettier / テストサイズ規約

本記事は **設計全体の俯瞰**を目的とする。各トピックの実装詳細は後掲のリンク先の記事を参照してほしい。

---

## リポジトリのディレクトリ構成

```
diary/                          ← リポジトリルート
├── .github/
│   └── workflows/
│       ├── ci.yml              ← push to main/develop
│       ├── ci-pr.yml           ← pull_request
│       ├── ci-nightly.yml      ← 毎朝 JST 03:00
│       ├── test-coverage.yml   ← カバレッジ計測（MySQL 付き）
│       └── copilot-setup-steps.yml
├── backend/                    ← Hono + Bun + MySQL ワークスペース
│   ├── src/
│   │   ├── index.ts
│   │   └── types/
│   │       └── eslint-plugins.d.ts   ← TS7016 回避用アンビエント宣言
│   ├── eslint.config.mts
│   ├── package.json
│   ├── bunfig.toml
│   └── .env.example
├── frontend/                   ← Next.js 16 + React 19 ワークスペース
│   ├── app/
│   │   └── api/mutator/
│   │       └── custom-instance.ts    ← Orval カスタム axios インスタンス
│   ├── eslint.config.mjs
│   ├── orval.config.ts
│   ├── vitest.config.ts
│   ├── playwright.config.ts
│   └── regconfig.json
├── .prettierrc                 ← 両ワークスペース共有の Prettier 設定
├── .prettierignore
└── .npmrc
```

`backend/` と `frontend/` はそれぞれ独立した `package.json` を持つが、ランタイム・パッケージマネージャーはどちらも Bun に統一している。

---

## 技術スタックの選定

| 領域 | バックエンド | フロントエンド |
|---|---|---|
| ランタイム | Bun | Bun (Next.js 経由) |
| HTTP / フレームワーク | Hono 4 | Next.js 16 (App Router) |
| データ層 | drizzle-orm + mysql2 | TanStack Query v5 + Orval |
| テスト | bun test (内蔵) | Vitest + Testing Library |
| E2E | — | Playwright |
| VRT | — | reg-suit + storycap |
| UI カタログ | — | Storybook 10 + MSW 2 |
| リンター | ESLint flat config (`.mts`) | ESLint flat config (`.mjs`) |
| フォーマッター | Prettier（ルート共有） | Prettier（ルート共有） |
| 型チェック | tsgo (`@typescript/native-preview`) | `tsc --noEmit` |

---

## 設計上の主要な決定事項

### 1. Bun をモノレポ全体のランタイムとパッケージマネージャーに採用

Node.js + npm/yarn の代わりに Bun を選んだ最大の理由は、**TypeScript をコンパイルなしで実行できる**ことだ。バックエンドでは `bun run --hot src/index.ts` で直接起動でき、CI にビルドステップが不要になる。

フロントエンドも `bun install` でパッケージを管理し、スクリプト実行は `bun run` で統一している。CI ではすべてのワークフローで `oven-sh/setup-bun@v1` + `bun install --frozen-lockfile` を使う。

```yaml
# 全ワークフロー共通のセットアップ
- uses: oven-sh/setup-bun@v1
  with:
    bun-version: latest
- run: bun install --frozen-lockfile
```

### 2. テストサイズ規約：small / medium / large

テストファイルの命名規則を **small / medium / large** で統一し、CI の各ステージが実行するテスト範囲を明示的に制御できる設計にした。

| サイズ | 対象 | CI での実行タイミング |
|---|---|---|
| small | ユニットテスト（外部依存なし） | push to main/develop、PR、ナイトリー |
| medium | インテグレーション（MSW 等でモック） | PR、ナイトリー |
| large | E2E、VRT | PR（E2E）、ナイトリー（全量）|

バックエンドは `bun test small.test` のようにグロブで絞り、フロントエンドは `vitest run --reporter=verbose small.test` で絞る。命名規則だけで制御するため、テストフレームワーク固有の設定が不要だ。

### 3. OpenAPI → Orval による型安全な API クライアント自動生成

フロントエンドは手書きのフェッチ関数を持たない。バックエンドの `@hono/zod-openapi` が生成する OpenAPI スキーマから、Orval が TanStack Query フックと Zod バリデーションスキーマを自動生成する設計にしている。

```
バックエンド (Hono + zod-openapi)
  → OpenAPI spec 公開
  → Orval が react-query hooks + zod schemas を生成
  → フロントエンドは生成コードを import して使う
```

スキーマが変わればフロントエンドの型エラーが即座に出るため、バックエンドとフロントエンドの型の乖離を開発時に検出できる。

カスタム axios インスタンスは `frontend/app/api/mutator/custom-instance.ts` に集約し、ベース URL の注入を一箇所で管理する。

```ts
// frontend/app/api/mutator/custom-instance.ts
const BACKEND_URL = process.env.NEXT_PUBLIC_BACKEND_URL ?? "http://localhost:3000";
const axiosInstance = create({ baseURL: BACKEND_URL });
```

`create` は `import { create } from "axios"` の名前付きインポートを使う（`eslint-plugin-import` の `import/no-named-as-default-member` ルール対応）。

### 4. ESLint flat config を両ワークスペースで整備

バックエンドは `eslint.config.mts`、フロントエンドは `eslint.config.mjs` を採用した。いずれも ESLint v9 の Flat Config 形式であり、従来の `.eslintrc` は使わない。

それぞれの主要プラグインは以下のとおり。

**バックエンド固有のプラグイン：**

| プラグイン | 目的 |
|---|---|
| `eslint-plugin-security` | API に特有のセキュリティリスク検出 |
| `eslint-plugin-drizzle` | `UPDATE`/`DELETE` の WHERE 句忘れをエラー化 |
| `eslint-plugin-jsdoc` | public 関数の JSDoc 必須化 |

**フロントエンド固有のプラグイン：**

| プラグイン | 適用対象 |
|---|---|
| `eslint-plugin-react` / `react-hooks` | React コンポーネント全般 |
| `eslint-plugin-jsx-a11y` | アクセシビリティ |
| `eslint-plugin-testing-library` / `@vitest/eslint-plugin` | テストファイルのみ |
| `eslint-plugin-storybook` | ストーリーファイルのみ |

**両ワークスペース共通：**

| プラグイン | 目的 |
|---|---|
| `typescript-eslint` | TypeScript 型安全ルール |
| `eslint-plugin-simple-import-sort` + `unused-imports` | import の整列・未使用削除 |
| `@stylistic/eslint-plugin` | コードスタイルの統一 |
| `@eslint-community/eslint-plugin-eslint-comments` | `eslint-disable` に説明を必須化 |
| `eslint-config-prettier` | ESLint と Prettier の競合解消（最後に配置）|

### 5. eslint-disable には必ず説明を書く

`@eslint-community/eslint-comments/require-description: "error"` を両ワークスペースに設定した。

```ts
// ❌ CI で落ちる
// eslint-disable-next-line camelcase

// ✅ CI を通る
// eslint-disable-next-line camelcase -- Geist_Mono は Next.js が snake_case で export している
```

`eslint-disable` コメントは説明なしで放置されると、後から誰も理由を知らない suppression が蓄積する。このルールはその蓄積を機械的に防ぐ。

### 6. Prettier 設定をリポジトリルートで一元管理

フォーマッター設定はリポジトリルートの `.prettierrc` に置き、バックエンドとフロントエンドの両方が参照する。

```json
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "all",
  "printWidth": 100,
  "proseWrap": "always"
}
```

これにより、バックエンドとフロントエンドで異なるフォーマットスタイルが混在することがない。生成ファイルや lock ファイルは `.prettierignore` で除外している。

### 7. VRT は Chromatic なし、reg-suit + storycap で自己完結

ビジュアルリグレッションテストには `reg-suit` を採用した。スクリーンショットは storycap で Storybook の全ストーリーを撮影し、`reg-publish-filesystem-plugin` でローカルファイルシステムに保存する。

```json
{
  "core": { "thresholdRate": 0 },
  "plugins": {
    "reg-publish-filesystem-plugin": { "publishDir": ".reg-snapshots" }
  }
}
```

外部サービスへの依存がなく、プライベートリポジトリでもコストが発生しない。VRT はナイトリーワークフローで実行し、差分が出た場合はアーティファクトにアップロードして確認できる。

---

## CI パイプライン設計

5つのワークフローがトリガーと目的で役割を分担している。

| ファイル | トリガー | 実行内容 | 目的 |
|---|---|---|---|
| `ci.yml` | push to main/develop | lint + typecheck + small テスト + build | 高速フィードバック |
| `ci-pr.yml` | pull_request | lint + small + medium + E2E + Storybook ビルド | PR の品質ゲート |
| `ci-nightly.yml` | cron (JST 03:00) + 手動 | 全テスト + storycap + VRT | 深夜の全量検証 |
| `test-coverage.yml` | workflow_dispatch | カバレッジ計測（MySQL サービス付き）| 品質指標の可視化 |
| `copilot-setup-steps.yml` | 変更時 / 手動 | bun install のみ | Copilot エージェント環境の事前準備 |

push to main/develop で実行される `ci.yml` が最も頻繁に動くため、small テスト（ユニットテスト）に絞ってフィードバックを速くしている。PR ではより広範な medium テストと E2E も実行してマージ前の品質を担保する。

---

## バックエンドの型チェック：@typescript/native-preview (tsgo)

```json
{
  "typecheck": "bun run --bun tsgo --noEmit"
}
```

バックエンドでは通常の `tsc` の代わりに `@typescript/native-preview`（`tsgo`）を使っている。TypeScript チームが開発中のネイティブ実装のプレビューパッケージで、`tsc` よりも高速に型チェックを完了できる。

**注意点：** これはプレビュー段階のパッケージであり、破壊的変更が入る可能性がある。CI で使う場合はバージョンピンニングの必要性を定期的に評価すること。

---

## 型定義のない ESLint プラグインへの対応

バックエンドの ESLint 設定ファイルは `.mts` 拡張子（TypeScript）で書かれているため、型定義を持たない ESLint プラグインを import すると `TS7016` エラーが発生する。

```
TS7016: Could not find a declaration file for module 'eslint-plugin-drizzle'.
TS7016: Could not find a declaration file for module 'eslint-plugin-security'.
```

これを解決するため、アンビエントモジュール宣言を追加した。

```ts
// backend/src/types/eslint-plugins.d.ts
declare module "eslint-plugin-drizzle";
declare module "eslint-plugin-security";
```

これは「このモジュールは存在するが型情報はない（`any` として扱う）」という意味の最小限の宣言だ。ESLint 設定ファイル専用のモジュールに対しては、`@types/xxx` をメンテするより軽量な対処法として有効。

---

## このブランチに含まれないもの

このブランチはスケルトン（骨格）に徹しており、以下は意図的に含めていない。

- 実際のアプリケーションルートやビジネスロジック
- データベーススキーマ定義
- 認証・認可の設定
- Docker / デプロイ設定
- テストファイル（インフラのみ構築済み）
- Orval 生成の API クライアントファイル（OpenAPI spec 未作成のため）
- VRT のベーススナップショット（初回ナイトリー実行後に撮影・コミット）

---

## クリーンアップ

このブランチでは不要になったファイルの削除も行っている。

- `agents/` ディレクトリ（4ファイル）：`.github/agents/` に移管済みのため削除
- 古い `pull-request/` ディレクトリ
- `frontend/CLAUDE.md`：Copilot エージェント体制に移行したため削除
- `docs/` ディレクトリ：`.github/` 以下に整理されたため削除

---

## 詳細記事へのリンク

各トピックの実装詳細は以下の記事で解説している。

| トピック | 記事 |
|---|---|
| バックエンド（Hono + Bun + MySQL + drizzle + ESLint）| `hono-bun-mysql-drizzle-backend-setup.md` |
| フロントエンド（Next.js + Vitest + Orval + reg-suit）| `フロントエンド開発環境セットアップ_Next.js_Vitest_Orval_reg-suit.md` |
| GitHub Actions（Bun 移行・CI 設計）| `GitHub_Actions_Bunへの移行とCI設計.md` |
| ESLint 品質改善（型エラー修正・eslint-disable 説明強制）| `ESLint品質改善_型エラー修正からeslint-disable説明強制まで.md` |

---

## まとめ

`settings` ブランチが整備したのは、次の 5 つの基盤だ。

1. **Bun で統一されたモノレポ**：バックエンド・フロントエンドとも Bun を使い、CI も `oven-sh/setup-bun@v1` で一本化。TypeScript のコンパイルステップなしで動作する。

2. **型安全な API 連携**：`@hono/zod-openapi` が OpenAPI spec を生成し、Orval がフロントエンドの TanStack Query フックと Zod スキーマを自動生成する。型の乖離を開発時に検出できる。

3. **テストサイズ規約による段階的 CI**：small / medium / large のファイル命名規則で CI のテスト範囲を制御する。push では small のみ、PR では medium まで、ナイトリーで全量という戦略が取れる。

4. **両ワークスペースで統一されたコード品質ルール**：`eslint-disable` への説明必須化（`require-description: "error"`）と Prettier のルート一元管理で、バックエンドとフロントエンドのコードスタイルが揃う。

5. **外部依存ゼロの VRT**：reg-suit + storycap でビジュアルリグレッションテストを自己完結させ、Chromatic や S3 などの外部サービスなしで運用できる。

アプリケーションコードは今後の Feature PR で追加していく。この基盤があれば、機能追加のたびに品質設定を見直す必要がなくなる。
