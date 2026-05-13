# infrastructure層統合テストで HonoApp・registry・in-memory repository を検証する

## 対象読者

- Hono で Clean Architecture のバックエンド実装をしている
- HTTP ルーティングとビジネスロジックの接合部を正しくテストしたい
- DI コンテナ（registry）の結線が正しく動作することを確認したい

## この記事が扱う範囲

- `createHonoApp()` での Hono ルーティング・controller 結線
- `createBackendRegistry()` による全層の結線確認
- `createInMemoryStorage()` の共有を通じた一貫性
- 404 / Content-Type ヘッダ / malformed body フォールバック
- `clearStorage()` による各テスト間の隔離

---

## 背景：infrastructure 層は複数層の接合部

infrastructure 層の責務：

- **DI コンテナ（registry）** — models → repositories → services → controllers を結線
- **HTTP ルーティング** — Hono アプリの route 定義
- **In-Memory ストレージ** — テスト用の shared storage
- **エラーハンドリング** — HTTP level での malformed body 対応

infrastructure 層は「薄い」と思われやすいが、**複数層が正しく接合されていることを実証**する責務がある。

---

## registry による依存性注入と結線

### createBackendRegistry の構造

```typescript
export function createBackendRegistry() {
  const storage = createInMemoryStorage();
  const appRepository = createInMemoryAppRepository(storage);
  const todoRepository = createInMemoryTodoRepository(storage);
  const appUsecase = createAppInteractor({
    appRepository,
    todoRepository,
  });
  const todoUsecase = createTodoInteractor({
    appRepository,
    todoRepository,
  });
  const appController = createAppController(appUsecase);
  const todoController = createTodoController(todoUsecase);
  const app = createHonoApp({
    appController,
    todoController,
  });

  return {
    app,
    clearStorage() {
      storage.clear();
    },
  };
}
```

**設計上のポイント：**

1. **共有 storage** — appRepository と todoRepository が同じ storage を参照 → cascade delete が動作
2. **単一責任** — 各層は自分の責務だけに集中し、registry が組み立て
3. **clearStorage() エクスポート** — テスト間の隔離を可能に

### registry テスト

```typescript
describe('BackendRegistry', () => {
  it('creates connected layers', () => {
    const registry = createBackendRegistry();
    expect(registry.app).toBeDefined();
    expect(typeof registry.clearStorage).toBe('function');
  });

  it('clearStorage clears both app and todo stores', async () => {
    const registry = createBackendRegistry();

    // Create data
    const res1 = await registry.app.request('http://localhost/api/v1/apps', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name: 'Test' }),
    });

    const created = await res1.json() as { data: { id: string } };
    const appId = created.data.id;

    // List should return 1 app
    const res2 = await registry.app.request('http://localhost/api/v1/apps');
    const list = await res2.json() as { data: unknown[] };
    expect(list.data).toHaveLength(1);

    // Clear storage
    registry.clearStorage();

    // List should return empty
    const res3 = await registry.app.request('http://localhost/api/v1/apps');
    const empty = await res3.json() as { data: unknown[] };
    expect(empty.data).toHaveLength(0);
  });
});
```

---

## HonoApp ルーティング・結線テスト

### createHonoApp の構造

```typescript
export function createHonoApp(dependencies: HonoAppDependencies) {
  const app = new Hono();

  app.get('/', c => c.text('Hello Hono!'));

  app.post('/api/v1/apps', async c =>
    toJsonResponse(
      c,
      await dependencies.appController.create(await readRequestBody(c)),
    ),
  );

  app.get('/api/v1/apps', async c =>
    toJsonResponse(c, await dependencies.appController.list()),
  );

  app.get('/api/v1/apps/:appId', async c =>
    toJsonResponse(
      c,
      await dependencies.appController.get(c.req.param('appId')),
    ),
  );

  // ... その他のルート

  return app;
}
```

**特徴：**

1. **薄い handler** — request parse / response mapping を `readRequestBody()` と `toJsonResponse()` に委譲
2. **controller への直接呼び出し** — HTTP フレームワークのロジックを入れない
3. **依存性注入** — controller が DI で渡される

### HonoApp テスト：ルーティング確認

```typescript
describe('HonoApp routing', () => {
  describe('POST /api/v1/apps', () => {
    it('calls appController.create and returns JSON', async () => {
      const { app } = buildApp();
      const res = await req(app, 'POST', '/api/v1/apps', { name: 'My App' });
      expect(res.status).toBe(201);
      expect(res.headers.get('content-type')).toMatch(/application\/json/);
    });
  });

  describe('GET /api/v1/apps', () => {
    it('calls appController.list', async () => {
      const { app } = buildApp();
      const res = await req(app, 'GET', '/api/v1/apps');
      expect(res.status).toBe(200);
    });
  });

  describe('GET /api/v1/apps/:appId', () => {
    it('calls appController.get with appId param', async () => {
      const { app } = buildApp();
      const created = await req(app, 'POST', '/api/v1/apps', { name: 'Test' });
      const data = await created.json() as { data: { id: string } };
      const appId = data.data.id;

      const res = await req(app, 'GET', `/api/v1/apps/${appId}`);
      expect(res.status).toBe(200);
    });
  });
});
```

---

## 404 と Content-Type テスト

### 404 Not Found

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

  it('POST /api/v1/unknown returns 404', async () => {
    const { app } = buildApp();
    const res = await req(app, 'POST', '/api/v1/unknown', {});
    expect(res.status).toBe(404);
  });
});
```

### Content-Type ヘッダ

```typescript
describe('Content-Type header', () => {
  it('GET /api/v1/apps returns application/json', async () => {
    const { app } = buildApp();
    const res = await req(app, 'GET', '/api/v1/apps');
    expect(res.headers.get('content-type')).toMatch(/application\/json/);
  });

  it('POST /api/v1/apps returns application/json', async () => {
    const { app } = buildApp();
    const res = await req(app, 'POST', '/api/v1/apps', { name: 'Test' });
    expect(res.headers.get('content-type')).toMatch(/application\/json/);
  });
});
```

---

## Malformed Body フォールバック

```typescript
describe('malformed request body', () => {
  it('POST /api/v1/apps with invalid JSON returns 400 or 422', async () => {
    const { app } = buildApp();
    const res = await app.request('http://localhost/api/v1/apps', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: 'not json',
    });
    // 400 Bad Request または 422 Unprocessable Entity
    expect([400, 422]).toContain(res.status);
  });

  it('POST /api/v1/apps with missing name returns 422', async () => {
    const { app } = buildApp();
    const res = await req(app, 'POST', '/api/v1/apps', {});
    expect(res.status).toBe(422);
  });
});
```

---

## readRequestBody と toJsonResponse

### readRequestBody のエラー処理

```typescript
async function readRequestBody(c: Context): Promise<unknown> {
  if (!c.req.valid('json')) {
    throw new AppError('VALIDATION_ERROR', 'Invalid JSON');
  }
  return await c.req.json().catch(() => ({}));
}
```

### toJsonResponse による HTTP 変換

```typescript
function toJsonResponse(c: Context, response: JsonHttpResponse): Response {
  return c.json(response.body, { status: response.status });
}
```

**ポイント：**

- controller が返す `JsonHttpResponse`（status + body）を、Hono Context 経由で HTTP Response に変換
- controller は HTTP フレームワークを知らない → テスト時に framework を使わずテスト可能

---

## in-memory repositories の Shared Storage

### Shared Storage テスト

```typescript
describe('Shared Storage between app and todo repos', () => {
  it('app deletion cascades to todos through shared storage', async () => {
    const { app, clearStorage } = buildApp();

    // Create app
    const appRes = await req(app, 'POST', '/api/v1/apps', { name: 'MyApp' });
    const appData = await appRes.json() as { data: { id: string } };
    const appId = appData.data.id;

    // Create 2 todos for this app
    await req(app, 'POST', `/api/v1/apps/${appId}/todos`, { title: 'Todo 1' });
    await req(app, 'POST', `/api/v1/apps/${appId}/todos`, { title: 'Todo 2' });

    // List todos for app - should have 2
    const listRes1 = await req(app, 'GET', `/api/v1/apps/${appId}/todos`);
    const list1 = await listRes1.json() as { data: unknown[] };
    expect(list1.data).toHaveLength(2);

    // Delete app
    await req(app, 'DELETE', `/api/v1/apps/${appId}`);

    // List todos for app - should be empty (cascade deleted)
    const listRes2 = await req(app, 'GET', `/api/v1/apps/${appId}/todos`);
    const list2 = await listRes2.json() as { data: unknown[] };
    expect(list2.data).toHaveLength(0);

    // App itself should be gone
    const getRes = await req(app, 'GET', `/api/v1/apps/${appId}`);
    expect(getRes.status).toBe(404);

    clearStorage();
  });
});
```

---

## infrastructure テストの成果

infrastructure 層統合テストにより：

1. **すべての層が正しく結線される** — registry から HTTP request まで、end-to-end で動作確認
2. **ルーティングが完全** — 定義されたすべてのエンドポイントが応答可能
3. **Shared Storage が機能する** — cascade delete など複数リポジトリの連携が成功
4. **HTTP boundary が正しい** — malformed body / 404 / Content-Type が期待通り
5. **clearStorage() による隔離** — テスト間の干渉がない

---

## まとめ

infrastructure 層テストは、models / repositories / services / controllers の「すべてが正しく接合されている」ことを唯一証明する層である。

特に **Shared Storage パターン**や **registry による DI** は、テストなしでは細部のバグが潜んでいる可能性が高い。

infrastructure テストを通じて、開発者は「自分の実装した各層が、ほかの層と正しく協調して動作している」という確信を得ることができる。
