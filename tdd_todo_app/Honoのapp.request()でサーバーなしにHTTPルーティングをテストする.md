# Hono の app.request() でサーバーを起動せずにルーティングをテストする

## 対象読者

- Hono で TypeScript バックエンドを構築している
- Express + supertest のような HTTP テストを Hono でも書きたい
- `app.fetch()` / `app.request()` の違いが気になっている

---

## Hono のテストは supertest が不要

Express + Node.js では HTTP テストに supertest が使われる。Hono は Web Standards（Fetch API）ベースで動作するため、`Hono` インスタンスに直接リクエストを送れる。サーバーを起動する必要がない。

```typescript
const res = await app.request('http://localhost/api/v1/apps');
```

`app.request()` は内部で `app.fetch()` を呼ぶラッパーで、`Request` オブジェクトまたは URL 文字列とオプションを受け取る。Node.js の `net` モジュールも、listen ポートも不要だ。

---

## テストのセットアップ

`hono-app.test.ts` では `buildApp()` ヘルパーでテスト用のアプリインスタンスを生成する。

```typescript
function buildApp() {
  const storage = createInMemoryStorage();
  const appRepository = createInMemoryAppRepository(storage);
  const todoRepository = createInMemoryTodoRepository(storage);
  const appUsecase = createAppInteractor({ appRepository, todoRepository });
  const todoUsecase = createTodoInteractor({ appRepository, todoRepository });
  const appController = createAppController(appUsecase);
  const todoController = createTodoController(todoUsecase);
  return {
    app: createHonoApp({ appController, todoController }),
    clearStorage: () => storage.clear(),
  };
}

function req(app, method: string, path: string, body?: unknown) {
  return app.request(`http://localhost${path}`, {
    method,
    headers: body !== undefined ? { 'Content-Type': 'application/json' } : undefined,
    body: body !== undefined ? JSON.stringify(body) : undefined,
  });
}
```

`createInMemoryStorage()` を共有することで、test 関数内の複数リクエストが同じ「仮想DB」を参照する。各テストケースで `buildApp()` を呼べばストアが新鮮な状態から始まる。

---

## 検証できること

### 1. ルーティングの結線確認

```typescript
describe('all app routes are wired', () => {
  it('POST /api/v1/apps creates an app (201)', async () => {
    const { app } = buildApp();
    const res = await req(app, 'POST', '/api/v1/apps', { name: 'A' });
    expect(res.status).toBe(201);
  });

  it('GET /api/v1/apps lists apps (200)', async () => {
    const { app } = buildApp();
    const res = await req(app, 'GET', '/api/v1/apps');
    expect(res.status).toBe(200);
  });

  it('GET /api/v1/apps/:id gets app (200)', async () => {
    const { app } = buildApp();
    const createRes = await req(app, 'POST', '/api/v1/apps', { name: 'B' });
    const { data } = await createRes.json() as { data: { id: string } };
    const res = await req(app, 'GET', `/api/v1/apps/${data.id}`);
    expect(res.status).toBe(200);
  });
});
```

コントローラーのロジックは別のテストで確認済みのため、ここでは「ルートがハンドラと正しく結線されているか」だけを確認する。ステータスコードの確認が主目的だ。

### 2. 404 ハンドリング

```typescript
describe('unknown routes', () => {
  it('GET /unknown returns 404', async () => {
    const { app } = buildApp();
    const res = await req(app, 'GET', '/unknown');
    expect(res.status).toBe(404);
  });

  it('GET /api/v1/unknown returns 404', async () => {
    const { app } = buildApp();
    const res = await req(app, 'GET', '/api/v1/unknown');
    expect(res.status).toBe(404);
  });
});
```

Hono はデフォルトで未登録ルートへのリクエストに 404 を返す。このテストが通ることで、誤ったパスでのルーティング設定がないことも確認できる。

### 3. Content-Type ヘッダーの確認

```typescript
describe('Content-Type header', () => {
  it('GET /api/v1/apps returns application/json Content-Type', async () => {
    const { app } = buildApp();
    const res = await req(app, 'GET', '/api/v1/apps');
    expect(res.headers.get('content-type')).toMatch(/application\/json/);
  });

  it('POST /api/v1/apps returns application/json Content-Type on error', async () => {
    const { app } = buildApp();
    const res = await req(app, 'POST', '/api/v1/apps', {});
    expect(res.headers.get('content-type')).toMatch(/application\/json/);
  });
});
```

成功レスポンスだけでなく、**エラーレスポンスも `application/json`** を返すことを確認している。クライアントが `Content-Type` を見てパース方法を決める場合、エラー時に `text/html` が返ると問題になる。

### 4. 不正な JSON ボディの扱い

```typescript
describe('malformed body handling', () => {
  it('POST with non-JSON body falls back to empty body (returns 422)', async () => {
    const { app } = buildApp();
    const res = await app.request('http://localhost/api/v1/apps', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: 'not-json',
    });
    expect(res.status).toBe(422);
  });
});
```

`Content-Type: application/json` を指定しながら JSON でないボディを送った場合の挙動を確認する。Hono アプリ側では `c.req.json()` が失敗したとき `{}` にフォールバックするため、バリデーションが `name is required` を返す。

```typescript
// src/infrastructure/hono-app.ts
async function readRequestBody(context: Context): Promise<unknown> {
  try {
    return await context.req.json();
  } catch {
    return {};  // ← パース失敗時は空オブジェクトとして扱う
  }
}
```

このフォールバック設計により、`500 Internal Server Error` の代わりに `422 Unprocessable Entity` を返せる。

---

## `helpers.ts` を使った HTTP テストユーティリティ

`tests/integrations/helpers.ts` には、より高レベルの HTTP ヘルパーが定義されている。これはアプリ全体をシングルトンで使う統合テスト向けだ。

```typescript
// src/tests/integrations/helpers.ts
import app, { clearStorage } from '../../index';

export const request = (method: string, path: string, body?: unknown) =>
  app.request(`http://localhost${path}`, {
    method,
    headers: body !== undefined ? { 'Content-Type': 'application/json' } : undefined,
    body: body !== undefined ? JSON.stringify(body) : undefined,
  });
```

`index.ts` からエクスポートされた `app` インスタンスを直接使う。E2E に近い統合テスト（`controllers/app-controller.test.ts` などのシナリオテスト）でこのヘルパーを使い、各テストの前後で `clearStorage()` を呼んでデータをリセットする。

---

## supertest との比較

| 観点 | supertest (Express) | app.request() (Hono) |
|---|---|---|
| サーバー起動 | 不要（内部で起動） | 不要 |
| 依存パッケージ | `supertest` が必要 | Hono 標準機能 |
| ポート競合 | なし | なし |
| ヘッダー確認 | `res.headers` | `res.headers.get()` |
| ボディ取得 | `res.body` | `await res.json()` |
| 環境依存 | Node.js 専用 | Bun / Node.js / Edge で動作 |

Hono の `app.request()` は Fetch API の `Response` を返すため、`await res.json()` でボディを取得する。レスポンスが消費済みになることに注意が必要だ（同じ `res` で `.json()` を2回呼べない）。

---

## まとめ

- Hono は `app.request()` でサーバーを起動せずに HTTP テストができる
- ルーティングの結線確認・404 ハンドリング・Content-Type・不正 JSON ボディの4種類を検証する
- エラーレスポンスも `application/json` を返すことを明示的にテストする
- 不正な JSON ボディは `{}` にフォールバックする設計で、パース失敗を 500 ではなく 422 に変換できる
