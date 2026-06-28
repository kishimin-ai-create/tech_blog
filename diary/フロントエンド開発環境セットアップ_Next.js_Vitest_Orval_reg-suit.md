# フロントエンド開発環境セットアップ：Next.js + Vitest + Orval + reg-suit の構成と設計判断

## 対象読者

- Next.js プロジェクトのゼロから環境を整えたいエンジニア
- OpenAPI からの型安全な API クライアント自動生成に興味がある人
- VRT（ビジュアルリグレッションテスト）を Chromatic を使わずに構築したい人
- テストサイズ規約（small / medium / large）を導入したい人

## 概要

Hono + Bun のバックエンドに続き、フロントエンドの開発環境を一から整備した。  
モノレポ構成の `frontend/` ディレクトリに Next.js 16 + React 19 を置き、テスト・VRT・API コード生成・Storybook まで一通りのツールチェーンを設定している。  
本記事では「何を選んだか」だけでなく「なぜそれを選んだか」「どう設定したか」を具体的なファイルと共に説明する。

---

## 技術スタックの選定

### フレームワーク：Next.js 16 + React 19

```json
"next": "16.2.6",
"react": "19.2.4",
"react-dom": "19.2.4"
```

App Router ベースで構成している。

### データフェッチ：TanStack Query + axios + Orval

データフェッチには **TanStack Query v5** を採用し、API 通信には **axios** を使う。  
ただし、手書きのフック関数は持たず、バックエンドが公開する OpenAPI スキーマから **Orval** でコードを自動生成する方針にした。

理由は明確で、手書きのフック関数はバックエンドの型変更に追従しにくく、型の不一致がランタイムエラーとして顕在化しやすい。Orval を使えばスキーマが変わった瞬間にフロントエンドの型エラーが出るため、開発中の安全ネットになる。

### テスティング：Vitest + Testing Library + Playwright

| ツール | 用途 |
|---|---|
| Vitest | ユニット・インテグレーションテスト |
| @testing-library/react | コンポーネントテスト |
| jsdom | ブラウザ環境のエミュレーション |
| Playwright | E2E テスト（Chromium + WebKit）|

Vitest は Vite 互換のため Next.js のエコシステムとの統合が自然で、ESM モジュールのサポートも良好。E2E は Playwright を選び、Chromium と WebKit の 2 ブラウザで実行する。

### VRT：reg-suit（Chromatic なし）

ビジュアルリグレッションテストには **reg-suit** を採用した。Chromatic は使わない（ユーザー要件）。

reg-suit はスナップショットを CI の artifact としてローカルに保存できるため、外部サービスへの依存がなく、プライベートリポジトリのコスト問題も発生しない。スクリーンショットの撮影には **storycap** を使い、Storybook のストーリーを全件キャプチャする。

### Storybook + MSW

```
@storybook/nextjs  ← Next.js App Router 対応フレームワーク
msw-storybook-addon ← MSW との統合
```

Storybook には **msw-storybook-addon** を組み込み、各ストーリーでバックエンド API をモックできるようにしている。Storybook の `preview.ts` で MSW を初期化する。

```ts
// frontend/.storybook/preview.ts
import { initialize, mswLoader } from "msw-storybook-addon";

initialize();

const preview: Preview = {
  loaders: [mswLoader],
  parameters: {
    nextjs: { appDirectory: true },
  },
};
```

---

## テストサイズ規約の強制

バックエンドと同じ **small / medium / large** のテストサイズ規約をフロントエンドにも適用している。  
ファイル名のパターンで分類し、実行スクリプトで個別に呼び出せる。

```ts
// frontend/vitest.config.ts
test: {
  include: ["**/*.{small,medium,large}.test.{ts,tsx}"],
}
```

```json
// package.json scripts
"test:small":    "vitest run --reporter=verbose small.test",
"test:medium":   "vitest run --reporter=verbose medium.test",
"test:large":    "vitest run --reporter=verbose large.test",
"test:coverage": "vitest run --coverage"
```

| サイズ | 対象 | 例 |
|---|---|---|
| small | ユニットテスト（外部依存なし）| 純粋関数、単体コンポーネント |
| medium | インテグレーション（MSW などでモック）| API フック、複数コンポーネント |
| large | E2E / VRT | Playwright, storycap |

CI の各ワークフローがどのサイズを実行するかを明示的に制御できるため、main ブランチへのプッシュでは small のみ、PR では small + medium + E2E、ナイトリーでは全サイズ + VRT という段階的な実行戦略が取れる。

### GitHub Actions 対応レポーター

Vitest のレポーターを環境変数で切り替えることで、GitHub Actions 上ではアノテーション付きの出力が得られる。

```ts
// frontend/vitest.config.ts
reporters: process.env.GITHUB_ACTIONS
  ? ["dot", "github-actions", "json"]
  : ["dot"],
outputFile: "test-result.json",
```

`github-actions` レポーターを指定すると、テスト失敗時に GitHub のプルリクエスト画面でファイル・行番号付きのアノテーションが表示される。また `outputFile` に `test-result.json` を指定しているため、CI でアーティファクトとしてアップロードできる。

---

## Orval による API クライアント自動生成

### 設定ファイル

```ts
// frontend/orval.config.ts
export default defineConfig({
  diary: {
    input: { target: "http://localhost:3000/openapi/v1.json" },
    output: {
      mode: "tags-split",
      target: "app/api/generated/diary.ts",
      schemas: "app/api/generated/model",
      client: "react-query",
      httpClient: "axios",
      mock: true,
      clean: true,
      formatter: "prettier",
      override: {
        mutator: {
          path: "app/api/mutator/custom-instance.ts",
          name: "customInstance",
        },
      },
    },
  },
  diaryZod: {
    input: { target: "http://localhost:3000/openapi/v1.json" },
    output: {
      mode: "tags-split",
      client: "zod",
      target: "app/api/generated/zod",
      fileExtension: ".zod.ts",
      formatter: "prettier",
    },
  },
});
```

`diary` ターゲットが TanStack Query + axios のフックを生成し、`diaryZod` ターゲットが Zod バリデーションスキーマを生成する。生成ファイルは ESLint の対象外にしている（`app/api/generated/**` を `globalIgnores` に追加）。

### カスタム axios インスタンス

Orval の `mutator` にカスタムインスタンスを指定することで、ベース URL の注入とレスポンスのアンラップを一箇所で管理している。

```ts
// frontend/app/api/mutator/custom-instance.ts
const BACKEND_URL = process.env.NEXT_PUBLIC_BACKEND_URL ?? "http://localhost:3000";
const axiosInstance = axios.create({ baseURL: BACKEND_URL });

export const customInstance = async <T>(config: AxiosRequestConfig): Promise<T> => {
  const response = await axiosInstance<T>(config);
  return response.data;
};
```

将来的に認証ヘッダーや共通エラーハンドリングを追加する場合もここに集約できる。

---

## reg-suit による VRT 設定

```json
// frontend/regconfig.json
{
  "core": {
    "workingDir": ".reg",
    "thresholdRate": 0,
    "concurrency": 4
  },
  "plugins": {
    "reg-simple-keygen-plugin": {
      "expectedKey": "snapshot",
      "actualKey": "snapshot"
    },
    "reg-publish-filesystem-plugin": {
      "publishDir": ".reg-snapshots"
    }
  }
}
```

`thresholdRate: 0` は 1px でも差異があれば失敗と判定する設定。  
`reg-publish-filesystem-plugin` でスナップショットをローカルファイルシステムに保存するため、S3 や Chromatic のような外部ストレージは不要。

実行フローは以下の通り：

1. `bun run build-storybook` — Storybook を静的ビルド
2. `bunx storycap` — 全ストーリーのスクリーンショットを撮影
3. `bun run vrt` (`reg-suit run`) — 前回スナップショットと比較

差分があると `.reg/` にレポートが生成される。CI ではナイトリーワークフローで実行し、差分発生時は artifact としてアップロードする。

---

## ESLint 設定の構成

フロントエンドの ESLint は **Flat Config** 形式で記述し、複数のプラグインを用途別にファイルパターンで分けて適用している。

| プラグイン | 適用対象 |
|---|---|
| `eslint-plugin-react` / `react-hooks` / `react-refresh` | `.ts` `.tsx` |
| `eslint-plugin-jsx-a11y` | `.ts` `.tsx`（アクセシビリティ）|
| `eslint-plugin-import` | `.ts` `.tsx`（import 順序）|
| `eslint-plugin-testing-library` / `@vitest/eslint-plugin` | テストファイルのみ |
| `eslint-plugin-storybook` | ストーリーファイル |
| `eslint-plugin-jsdoc` | `publicOnly: true` で公開 API に JSDoc を強制 |
| `eslint-plugin-unused-imports` | 未使用 import を自動検出 |

重要な設計判断として、`app/api/generated/**` は ESLint 対象外にしている。自動生成ファイルに lint を走らせると Orval の出力が変わるたびに lint エラーが出る可能性があるため。

型チェックが必要な TypeScript ルール（`tseslint.configs.recommendedTypeChecked`）は設定ファイルや E2E ファイルには適用しない。これらは TypeScript のプロジェクトサービスコンテキスト外で実行されることがあり、偽エラーを避けるため `tseslint.configs.disableTypeChecked` を上書きしている。

---

## Prettier の共有設定

Prettier の設定ファイルはリポジトリルートに置き、バックエンドとフロントエンドで共有している。

```json
// .prettierrc (リポジトリルート)
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "all",
  "printWidth": 100,
  "proseWrap": "always"
}
```

バックエンド（Hono + Bun）とフロントエンド（Next.js）でコードスタイルが揃う。ESLint との競合は `eslint-config-prettier` を最後に適用して解決している。

---

## Playwright E2E テスト設定

```ts
// frontend/playwright.config.ts
projects: [
  { name: "chromium", use: { ...devices["Desktop Chrome"] } },
  { name: "webkit",   use: { ...devices["Desktop Safari"] } },
],
webServer: {
  command: "bun run dev",
  url: "http://localhost:3000",
  reuseExistingServer: !process.env.CI,
},
```

E2E テストは Chromium と WebKit の 2 環境で実行する。`webServer` の設定により、テスト実行時に自動でサーバーが起動するため、テストの実行コマンドを 1 つ叩くだけで済む。CI 環境では `reuseExistingServer: false` になるため、毎回クリーンなサーバーが起動する。

---

## まとめ

| 領域 | 選択 | 理由 |
|---|---|---|
| フレームワーク | Next.js 16 / React 19 | App Router、最新安定版 |
| API クライアント | Orval + TanStack Query + axios | OpenAPI スキーマから型安全に自動生成 |
| ユニット/インテグレーション | Vitest + Testing Library + jsdom | Vite 互換、高速 |
| E2E | Playwright | Chromium + WebKit の 2 ブラウザ対応 |
| VRT | reg-suit + storycap | Chromatic なし、外部依存なし |
| UI カタログ | Storybook + MSW | API モック付きストーリー |
| コードスタイル | ESLint Flat Config + Prettier | shadcn/ui・Biome は不採用 |

テストサイズ規約（small / medium / large）をファイル命名で強制することで、CI の各ステージが「何を実行するか」を明示的にコントロールできる構成になっている。バックエンドとフロントエンドで同じ規約を使うため、チーム全体でテスト分類の認識を揃えやすい。
