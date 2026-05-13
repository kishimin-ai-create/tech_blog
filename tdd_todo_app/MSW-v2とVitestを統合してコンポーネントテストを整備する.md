# MSW v2 と Vitest を統合してコンポーネントテストを整備する

## 対象読者

- React + Vitest でコンポーネントテストを書きたいが、API 呼び出しをどうモックするか迷っている人
- MSW v2 を導入したが Vitest との連携方法がわからない人
- `@testing-library/react` で DOM テストをセットアップしたい人

## この記事で扱うこと / 扱わないこと

**扱う**
- MSW v2 の `setupServer` を Vitest のライフサイクルに組み込む方法
- jsdom 環境の設定と `@testing-library/jest-dom` のセットアップ
- per-test でモックハンドラーをオーバーライドするパターン

**扱わない**
- orval による MSW ハンドラーの自動生成（それは[前の記事](./orvalでOpenAPI定義書から型・モック・TanStackQueryフックを自動生成する.md)で扱う）
- Storybook との連携
- ブラウザ向け MSW（`setupWorker`）の使い方

---

## 背景

前の工程で [orval](https://orval.dev/) を設定し、OpenAPI 定義書から MSW ハンドラーを自動生成できるようにした。生成された `src/api/generated/index.msw.ts` には `getTDDTodoAppAPIMock()` という関数があり、全エンドポイントのハンドラーをまとめて返す。

しかしこれを実際のテストで使うには、MSW のサーバーを立ち上げ Vitest のテストライフサイクルに連動させる必要がある。また、既存のテスト設定は DOM を使っていなかったため、コンポーネントテストに必要な jsdom 環境と `@testing-library/react` もまだ入っていなかった。

今回の作業はこの「最後の一マイル」をつなぐ設定をまとめて行ったものだ。

---

## インストール

```bash
npm install -D @testing-library/react jsdom
```

`msw` と `@testing-library/jest-dom`、`@testing-library/user-event` はすでに `package.json` に含まれていたため追加は不要だった。

---

## 設定の全体像

変更したファイルは4つだ。

1. `src/test/server.ts` — MSW サーバーのインスタンス定義
2. `src/test/setup.ts` — Vitest グローバルセットアップ
3. `vite.config.ts` — Vitest の環境設定
4. （`package.json` への devDependency 追加は上述）

---

## `src/test/server.ts`

```ts
import { setupServer } from 'msw/node';

import { getTDDTodoAppAPIMock } from '../api/generated/index.msw';

/**
 * MSW server shared across all Vitest tests.
 * Handlers are auto-generated from the OpenAPI spec via orval.
 */
export const server = setupServer(...getTDDTodoAppAPIMock());
```

### ポイント

- `msw/node` から `setupServer` をインポートする（ブラウザ用の `setupWorker` ではない）。Vitest は Node.js 上で動くため `msw/node` を使う。
- `getTDDTodoAppAPIMock()` は orval が生成した全ハンドラーの配列を返す。スプレッドで展開して渡す。
- このファイルはサーバーの**インスタンスを作るだけ**で起動はしない。起動・停止はセットアップファイルで行う。

---

## `src/test/setup.ts`

```ts
import '@testing-library/jest-dom/vitest';
import { afterAll, afterEach, beforeAll } from 'vitest';

import { server } from './server';

beforeAll(() => {
  server.listen({ onUnhandledRequest: 'warn' });
});

afterEach(() => {
  server.resetHandlers();
});

afterAll(() => {
  server.close();
});
```

### 3つのライフサイクルフックの役割

| フック | タイミング | 処理 |
|--------|-----------|------|
| `beforeAll` | 全テスト実行前に1回 | MSW サーバーを起動 |
| `afterEach` | 各テスト終了後 | `server.use()` で追加したハンドラーをリセット |
| `afterAll` | 全テスト終了後に1回 | MSW サーバーを停止 |

`resetHandlers()` を `afterEach` で呼ぶのは重要だ。特定テストで `server.use()` によってハンドラーを追加・上書きした場合、それを次のテストに持ち越さないためのリセットだ。

### `onUnhandledRequest: 'warn'`

定義書に載っていないエンドポイントへのリクエストが発生したとき、`'error'` にするとテストが落ちる。`'warn'` にすることで、新しいエンドポイントを追加した直後でも既存テストが壊れない。

### `@testing-library/jest-dom/vitest`

`@testing-library/jest-dom` v6 系は Vitest 専用エントリポイントとして `/vitest` を提供している。これをインポートすると `expect(element).toBeInTheDocument()` などのカスタムマッチャーが Vitest の `expect` に追加される。

`/vitest` ではなく `@testing-library/jest-dom` をそのままインポートすると型が合わないことがあるため、Vitest 環境では必ず `/vitest` エントリを使う。

---

## `vite.config.ts`（Vitest 設定）

```ts
test: {
  environment: 'jsdom',
  setupFiles: ['src/test/setup.ts'],
  include: ['src/**/*.{test,spec}.{ts,tsx}'],
},
```

### `environment: 'jsdom'`

デフォルトでは Vitest は Node.js 環境で動く。`jsdom` を指定することで `document`、`window`、`HTMLElement` などのブラウザ API が使える状態になり、`@testing-library/react` の `render()` が動作するようになる。

`jsdom` は Node.js でブラウザ環境をシミュレートするため、インストールが必要だ（`npm install -D jsdom`）。

### `setupFiles`

ここに指定したファイルは各テストファイルの実行前に読み込まれる。`src/test/setup.ts` を指定することで、MSW のライフサイクルフックとカスタムマッチャーが全テストファイルで自動的に有効になる。

---

## テストでのモックオーバーライド

デフォルトでは `getTDDTodoAppAPIMock()` が返すハンドラーがすべてのエンドポイントに適用される（ランダムな faker データを返す）。

特定テストで特定のレスポンスを返したい場合は `server.use()` を使う。

```ts
import { server } from 'src/test/server';
import { getGetApiV1AppsMockHandler } from 'src/api/generated/index.msw';

test('アプリ一覧が表示される', async () => {
  server.use(
    getGetApiV1AppsMockHandler({
      success: true,
      data: [{ id: 'app-1', name: 'My App', createdAt: '...', updatedAt: '...' }],
    }),
  );

  // ...render + assert
});
```

`afterEach` で `server.resetHandlers()` が呼ばれるため、この上書きは当該テストスコープのみに限定される。

---

## まとめ

| 役割 | ファイル |
|------|---------|
| MSW サーバーのインスタンス | `src/test/server.ts` |
| Vitest ライフサイクル連携 | `src/test/setup.ts` |
| jsdom 環境 + セットアップ登録 | `vite.config.ts` |

この構成を整えると、コンポーネントテストで次のことが可能になる。

- **デフォルト**: orval 生成ハンドラーが全エンドポイントをカバーするため、API を意識せずコンポーネントをレンダリングできる
- **オーバーライド**: `server.use()` で特定テストのシナリオに合わせたレスポンスを返せる
- **型安全**: ハンドラーのオーバーライドレスポンスも OpenAPI 由来の型で検査される

バックエンドの OpenAPI 定義書を更新 → `npm run generate` で再生成 → フロントエンドの型とモックが同期、というループが確立した。
