# orval で OpenAPI 定義書から型・モック・TanStack Query フックを自動生成する

## 対象読者

- React + TypeScript フロントエンドで REST API と通信しているが、型を手書きしていて辛い人
- OpenAPI 定義書はあるのに、フロントエンドの型と乖離しがちで困っている人
- TanStack Query と MSW を組み合わせてテストしやすい構成を作りたい人

## この記事で扱うこと / 扱わないこと

**扱う**
- orval の設定（`orval.config.ts`）の構成と各オプションの意味
- カスタム fetch mutator の実装
- `npm run generate` で生成されるファイルの種類と用途
- ESLint / TypeScript との統合でつまずきやすいポイント

**扱わない**
- OpenAPI 定義書そのものの書き方
- TanStack Query の使い方詳細
- MSW のモックサーバーをテストで使う方法（それは別記事で扱う）

---

## 背景

このプロジェクトでは Hono バックエンドに対して hono-openapi で OpenAPI 定義書を自動生成している。定義書は `docs/spec/backend/openapi-generated-pretty.json` として管理されており、エンドポイント数は `/api/v1/apps` と `/api/v1/apps/{appId}/todos` を合わせた 10 本のフル CRUD だ。

フロントエンドはまだ実装が始まったばかりだったが、今後 API と通信するコードを書き始めるにあたって **型の手書きは避けたい** という方針があった。また、**コンポーネントテストで API を差し替えやすくする**ために MSW ハンドラーも同じ定義書から生成したかった。

そこで [orval](https://orval.dev/) を導入し、一つのコマンドで以下3種類の成果物を生成する仕組みを整えた。

| 生成物 | ファイル | 用途 |
|--------|---------|------|
| TypeScript 型定義 | `src/api/generated/models/` | リクエスト・レスポンスの型 |
| TanStack Query フック | `src/api/generated/index.ts` | コンポーネントからの API 呼び出し |
| MSW ハンドラー | `src/api/generated/index.msw.ts` | テスト用モック |

---

## 設定の全体像

### `orval.config.ts`

```ts
import { defineConfig } from 'orval';

export default defineConfig({
  todoApp: {
    input: {
      target: '../docs/spec/backend/openapi-generated-pretty.json',
    },
    output: {
      mode: 'split',
      target: 'src/api/generated/index.ts',
      schemas: 'src/api/generated/models',
      client: 'react-query',
      httpClient: 'fetch',
      mock: {
        type: 'msw',
      },
      clean: true,
      override: {
        mutator: {
          path: 'src/api/client.ts',
          name: 'customFetch',
        },
      },
    },
  },
});
```

各オプションの意味を解説する。

#### `mode: 'split'`

生成物の分割方法を指定する。`'split'` にすると、API フック群は `index.ts` 一ファイルにまとまり、型定義は `models/` 配下にファイルごとに分割される。

OpenAPI タグが定義されている場合は `'tags-split'` を使うとタグ別にサブディレクトリが作られるが、このプロジェクトの定義書にはタグが付いていないため `'split'` を選んだ。

#### `client: 'react-query'`

生成するフックのクライアントタイプ。TanStack Query の `useQuery` / `useMutation` を使ったフックを生成する。

#### `httpClient: 'fetch'`

HTTP 通信に使うクライアント。`'axios'` と `'fetch'` が選べる。ここでは標準の `fetch` を選んでいる。

#### `mock: { type: 'msw' }`

MSW ハンドラーも同時生成するための設定。`index.msw.ts` というファイルが `index.ts` と並べて生成される。

#### `mutator`

orval が生成するコードは HTTP 呼び出しを特定の関数に委譲できる。`mutator` にカスタム関数を指定することで、ベース URL の差し込みやエラーハンドリングなどを一箇所で制御できる。

---

### `src/api/client.ts`（カスタム fetch mutator）

```ts
const BASE_URL: string = (import.meta.env['VITE_API_BASE_URL'] as string) ?? '';

/**
 * Custom fetch wrapper used by orval-generated API client.
 * Prepends VITE_API_BASE_URL to every request and handles JSON parsing.
 */
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

`VITE_API_BASE_URL` を環境変数として注入することで、開発環境とプロダクション環境でベース URL を切り替えられる。開発環境では Vite の proxy（後述）を使うため空文字でよく、デフォルト `''` にしている。

`response.ok` が false のとき（4xx / 5xx）は JSON ボディをそのまま `throw` する。orval が生成するフック側でこのエラーをキャッチできる。

---

## 生成されるファイル

### `src/api/generated/index.ts`（TanStack Query フック群）

```ts
// 生成例: Apps 一覧取得フック
export const useGetApiV1Apps = <
  TData = Awaited<ReturnType<typeof getApiV1Apps>>,
  TError = GetApiV1Apps500,
>(
  options?: { query?: UseQueryOptions<...>, request?: SecondParameter<typeof customFetch> }
): UseQueryResult<TData, TError> & { queryKey: DataTag<QueryKey, TData, TError> }
```

全エンドポイント分の `useQuery` / `useMutation` フックが生成される。ジェネリクスによって戻り値の型とエラー型が正確に推論される。

### `src/api/generated/index.msw.ts`（MSW ハンドラー）

```ts
// 生成例: Apps 一覧のモックハンドラー
export const getGetApiV1AppsMockHandler = (
  overrideResponse?: GetApiV1Apps200 | ((info: Parameters<Parameters<typeof http.get>[1]>[0]) => Promise<GetApiV1Apps200> | GetApiV1Apps200),
) => {
  return http.get('*/api/v1/apps', async () => {
    return HttpResponse.json(
      overrideResponse !== undefined
        ? (typeof overrideResponse === 'function' ? await overrideResponse(...) : overrideResponse)
        : getGetApiV1AppsResponseMock(),
      { status: 200 }
    );
  });
};

// 全ハンドラーをまとめて返すアグリゲート関数
export const getTDDTodoAppAPIMock = () => [
  getPostApiV1AppsMockHandler(),
  getGetApiV1AppsMockHandler(),
  // ...
];
```

`@faker-js/faker` を使ってレスポンスをランダム生成する。各ハンドラーは `overrideResponse` を受け取るため、特定テストで特定のレスポンスを返すよう上書きできる。

### `src/api/generated/models/`（型定義）

エンドポイントごと・ステータスコードごとにファイルが分かれて生成される。

```
models/
  getApiV1Apps200.ts        ← 200 レスポンス全体の型
  getApiV1Apps200DataItem.ts ← data 配列の要素型
  getApiV1Apps500.ts
  getApiV1Apps500Error.ts
  ...
```

---

## 周辺設定

### `package.json` にスクリプトを追加

```json
{
  "scripts": {
    "generate": "orval"
  }
}
```

`npm run generate` で定義書から全ファイルが再生成される。OpenAPI 定義書が更新されたタイミングで実行する。

### Vite dev proxy

開発時にブラウザからバックエンドへのリクエストが CORS エラーにならないよう、Vite の dev server proxy を設定する。

```ts
// vite.config.ts
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:3000',
      changeOrigin: true,
    },
  },
},
```

これにより開発中は `VITE_API_BASE_URL` を設定しなくても `/api/...` へのリクエストが `localhost:3000` に転送される。

---

## つまずきポイント

### `@faker-js/faker` が必要

orval が MSW 用モックコードを生成すると、`@faker-js/faker` を import する。しかしこれは orval の `peerDependencies` に明示されておらず、インストールしていないと TypeScript エラーが出る。

```bash
npm install -D @faker-js/faker
```

### ESLint の `globalIgnores` に生成ファイルを追加

生成されたコードは JSDoc ルール等で lint エラーになる。`eslint.config.js` の `globalIgnores` に追加する。

```js
globalIgnores(['src/api/generated/**'])
```

### `tsconfig.node.json` に `orval.config.ts` を追加

ESLint の project service は `tsconfig.node.json` を参照する。`orval.config.ts` を `include` に追加しないと「project service の範囲外」エラーが出る。

```json
{
  "include": ["vite.config.ts", "orval.config.ts"]
}
```

---

## まとめ

orval を導入することで、OpenAPI 定義書という **単一の信頼できる情報源** からフロントエンドに必要な型・フック・モックを一括生成できるようになった。

| 以前 | 以後 |
|------|------|
| 型を手書き | `npm run generate` で自動生成 |
| 型の乖離を発見しにくい | バックエンド変更 → 定義書更新 → 生成 → 型エラーで即検知 |
| MSW ハンドラーも手書き | 定義書から自動生成、オーバーライドで柔軟にカスタマイズ |

次のステップとして、生成した MSW ハンドラーを Vitest に統合し、コンポーネントテストで実際に使える状態にする。それは[MSW v2 と Vitest を統合してコンポーネントテストを整備する](./MSW-v2とVitestを統合してコンポーネントテストを整備する.md)で取り上げる。
