# OpenAPI定義からフロントエンドのAPI層を自動構築する — orval + MSW v2 + Vitestの連携設計

## 対象読者

- React + TypeScript フロントエンドで REST API 型を手書きしており、バックエンドとの乖離に悩んでいる人
- OpenAPI 定義書はあるのに、フロントエンドのモック整備が後回しになっている人
- TanStack Query と MSW v2 を組み合わせたテスト環境を整えたい人

---

## 背景

このプロジェクトでは Hono バックエンドに対して hono-openapi で OpenAPI 定義書を自動生成している。定義書は `docs/spec/backend/openapi-generated-pretty.json` に出力されており、`/api/v1/apps` と `/api/v1/apps/{appId}/todos` を合わせた 10 本のフル CRUD エンドポイントをカバーしている。

フロントエンドの実装を始める前に、次の3つの問題を解決する必要があった。

1. **型の二重管理** — バックエンドの DTO を手書きで複製すると、スキーマ変更のたびに同期コストが発生する
2. **テスト用モックの整備** — コンポーネントテストで API をリアルに差し替えるには、スキーマと一致したハンドラーが必要
3. **TanStack Query フックの手書きコスト** — `useQuery` / `useMutation` の定型コードは量が多い

この3つをまとめて解決するために [orval](https://orval.dev/) を導入し、一つのコマンドで型・TanStack Query フック・MSW ハンドラーを生成する仕組みを整えた。

---

## orval による自動生成の全体像

```
docs/spec/backend/openapi-generated-pretty.json
        │
        │  npx orval
        ▼
frontend/src/api/generated/
├── models/          ← 80 個の DTO 型ファイル
├── index.ts         ← TanStack Query フック
└── index.msw.ts     ← MSW v2 ハンドラーファクトリ
```

`orval.config.ts` の設定がこの出力を制御する。

```ts
// frontend/orval.config.ts
import { defineConfig } from 'orval';

export default defineConfig({
  todoApp: {
    input: {
      target: '../docs/spec/backend/openapi-generated-pretty.json',
    },
    output: {
      mode: 'split',           // DTO ごとにファイルを分割
      target: 'src/api/generated/index.ts',
      schemas: 'src/api/generated/models',
      client: 'react-query',   // TanStack Query フックを生成
      httpClient: 'fetch',
      mock: {
        type: 'msw',           // MSW v2 ハンドラーを生成
      },
      clean: true,             // 再生成時に古いファイルを削除
      override: {
        mutator: {
          path: 'src/api/client.ts',
          name: 'customFetch', // 全フックが使うカスタム fetch を指定
        },
      },
    },
  },
});
```

`mode: 'split'` を選ぶと、生成されたモデル型が 1 ファイル 1 型に分割される。大規模 API では `index.ts` が肥大化するのを防げるため、このプロジェクトでは split モードを採用した。

---

## カスタム fetch ラッパー

生成されたフックは素の `fetch` ではなく、`override.mutator` で指定した `customFetch` を呼び出す。

```ts
// frontend/src/api/client.ts
const BASE_URL: string = (import.meta.env['VITE_API_BASE_URL'] as string) ?? '';

export const customFetch = async <T>(
  url: string,
  options?: RequestInit,
): Promise<T> => {
  const response = await fetch(`${BASE_URL}${url}`, options);

  if (!response.ok) {
    throw (await response.json()) as unknown;
  }

  return response.json() as Promise<T>;
};
```

このラッパーは2つの役割を担う。

| 役割 | 実装 |
|------|------|
| ベース URL の注入 | `VITE_API_BASE_URL` 環境変数を先頭に付与 |
| エラー処理の統一 | 非 OK レスポンスでは JSON ボディをそのままスローする |

`VITE_API_BASE_URL` を空文字のままにしておけば、Vite の dev server プロキシ（`/api → http://localhost:3000`）がリクエストを転送するため、ローカル開発中は特別な設定なしで動く。

---

## MSW v2 + Vitest 統合

### サーバーの生成

orval が生成した `getTDDTodoAppAPIMock()` は全エンドポイントのデフォルトハンドラーを返す関数だ。これを MSW の `setupServer` に渡すだけで、スキーマに準拠したモックサーバーが完成する。

```ts
// frontend/src/test/server.ts
import { setupServer } from 'msw/node';
import { getTDDTodoAppAPIMock } from '../api/generated/index.msw';

export const server = setupServer(...getTDDTodoAppAPIMock());
```

このファイルは Vitest の全テストから `import` される共有インスタンスとなる。サーバーを分離しておくことで、テストファイルが `server.use(...)` で特定テスト用のハンドラーに差し替えやすくなる。

### Vitest ライフサイクルへの組み込み

```ts
// frontend/src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { afterAll, afterEach, beforeAll } from 'vitest';
import { server } from './server';

beforeAll(() => {
  server.listen({ onUnhandledRequest: 'warn' });
});

afterEach(() => {
  server.resetHandlers(); // テストごとにハンドラーをリセット
});

afterAll(() => {
  server.close();
});
```

`afterEach` で `resetHandlers()` を呼ぶことで、あるテストで `server.use(...)` した一時的なオーバーライドが次のテストに漏れない。`onUnhandledRequest: 'warn'` は、モックされていないリクエストが発生したときに警告を出して気づかせる。

### Vite の設定

```ts
// frontend/vite.config.ts（抜粋）
export default defineConfig({
  // ...
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },
  test: {
    environment: 'jsdom',
    setupFiles: ['src/test/setup.ts'],
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
  },
});
```

`environment: 'jsdom'` でブラウザ相当の DOM 環境を使い、`setupFiles` に `setup.ts` を指定することで全テスト実行前に MSW サーバーが自動起動する。

---

## OpenAPI 定義書が変わったときの運用

生成ファイルはリポジトリにコミットされている（`.gitignore` から除外している）。これにより、コード生成ツールをインストールせずに型を import できる。

OpenAPI 定義書が更新されたら以下を実行する。

```bash
cd frontend
npx orval          # 再生成
npm run typecheck  # 型エラーがないか確認
npm run test       # MSW ハンドラーと整合性を確認
```

型エラーが出た場合は、生成されたフックの呼び出し箇所が古い型を参照しているサインだ。`typecheck` が 0 エラーになるまで修正する。

---

## この設計で得られるもの

| 問題 | 解決策 |
|------|--------|
| バックエンド型との乖離 | `npm run typecheck` で即座に検出できる |
| テスト用モックの陳腐化 | `npx orval` で自動再生成される |
| TanStack Query の定型コード | 生成フックを import するだけ |
| モック忘れによるネットワークエラー | `onUnhandledRequest: 'warn'` が警告を出す |

---

## まとめ

- **orval** は OpenAPI 定義書から型・TanStack Query フック・MSW ハンドラーを一括生成する
- `customFetch` ラッパーをmutatorとして登録することで、全フックが環境変数ベースのベース URL とエラー処理を自動的に使う
- **MSW v2 + Vitest** の組み合わせでは、`setupServer` を共有インスタンスとして分離し、`setupFiles` から起動するのが定石
- 生成ファイルをコミットしておくことで、CI でのコード生成ステップが不要になり、型チェックで乖離を検出できる

フロントエンドの API 層は「実装するもの」ではなく「定義書から生成するもの」という発想の転換が、長期的なメンテナンスコストを下げる。
