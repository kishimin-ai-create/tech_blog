# controllers層統合テストで HTTP ステータスと DTO 変換を検証する

## 対象読者

- Clean Architecture で controller 層を実装している
- HTTP ステータスコード（201 / 200 / 422 / 409 / 404）の使い分けを確実にしたい
- DTO 変換（内部エンティティ → JSON スキーマ）をテストしたい

## この記事が扱う範囲

- `AppController` / `TodoController` の全操作テスト
- HTTP status code（201 / 200 / 422 / 409 / 404）の検証
- `presentApp()` / `presentTodo()` による DTO 変換
- `deletedAt` がレスポンスに含まれないことの検証
- エラーレスポンスの shape

---

## 背景：controller 層はビジネスロジックと HTTP の仲介役

controller 層の責務：

- **リクエストパース** — JSON body を usecase input に変換
- **エラーハンドリング** — AppError → HTTP status code
- **DTO 生成** — エンティティ → レスポンス JSON
- **ステータスコード決定** — 201 / 200 / 422 / 409 / 404 の使い分け

テストなしでは、以下の問題が検出されない：

- deletedAt がレスポンスに漏れている
- エラーコードとステータスコードのマッピングが間違っている
- 成功時のステータスが 200 だが 201 が必要な場合

---

## AppController テスト構成

```typescript
export type AppController = {
  create(body: unknown): Promise<JsonHttpResponse>;
  list(): Promise<JsonHttpResponse>;
  get(appId: string): Promise<JsonHttpResponse>;
  update(appId: string, body: unknown): Promise<JsonHttpResponse>;
  delete(appId: string): Promise<JsonHttpResponse>;
};

export type JsonHttpResponse = {
  status: number;
  body: ApiResponseBody;
};

export type ApiResponseBody = {
  data: unknown;
  success: boolean;
  error?: ErrorBody;
};
```

controller は純粋な値オブジェクト（`JsonHttpResponse`）を返すため、HTTP フレームワーク（Hono / Express）を使わずテスト可能。

---

## Create 操作のテスト

### 成功系：201 + DTO 検証

```typescript
describe('create', () => {
  it('returns 201 with success:true and app DTO on valid input', async () => {
    const res = await ctx.controller.create({ name: 'My App' });
    expect(res.status).toBe(201);
    expect(res.body.success).toBe(true);
    expect(res.body.data).toMatchObject({
      id: expect.any(String),
      name: 'My App',
      createdAt: expect.any(String),
      updatedAt: expect.any(String),
    });
  });

  it('DTO does not include deletedAt', async () => {
    const res = await ctx.controller.create({ name: 'App' });
    expect((res.body.data as Record<string, unknown>)).not.toHaveProperty('deletedAt');
  });
});
```

**テスト設計のポイント：**

1. **ステータスコード 201** — 作成操作は 201 Created
2. **success フラグ** — 成功時は `true`
3. **DTO shape** — id / name / createdAt / updatedAt を含む
4. **deletedAt の隠蔽** — DTO には含めない（内部フィールド）

内部エンティティは `deletedAt` を持つが、**HTTP レスポンス（DTO）には含めない**というスキーマ分離が重要。

### バリデーションエラー：422

```typescript
it('returns 422 when name is missing', async () => {
  const res = await ctx.controller.create({});
  expect(res.status).toBe(422);
  expect(res.body.success).toBe(false);
  expect(res.body.error?.code).toBe('VALIDATION_ERROR');
});

it('returns 422 when name is not a string', async () => {
  const res = await ctx.controller.create({ name: 42 });
  expect(res.status).toBe(422);
  expect(res.body.error?.code).toBe('VALIDATION_ERROR');
});

it('returns 422 when name is an empty string or whitespace', async () => {
  const res = await ctx.controller.create({ name: '   ' });
  expect(res.status).toBe(422);
});
```

**422 Unprocessable Entity の使い分け：**

- name フィールドがない
- name が string でない
- name が空文字列または空白のみ

---

## 一意制約違反：409

```typescript
it('returns 409 when the name is already taken', async () => {
  await ctx.controller.create({ name: 'Dup' });
  const res = await ctx.controller.create({ name: 'Dup' });
  expect(res.status).toBe(409);
  expect(res.body.success).toBe(false);
  expect(res.body.error?.code).toBe('CONFLICT');
});

it('first create with name succeeds, second fails with 409', async () => {
  const res1 = await ctx.controller.create({ name: 'Unique' });
  expect(res1.status).toBe(201);

  const res2 = await ctx.controller.create({ name: 'Unique' });
  expect(res2.status).toBe(409);
});
```

**409 Conflict の条件：**

- app name が既存（active）のレコードと一致
- `AppError('CONFLICT', ...)` がスロー
- HTTP 409 ステータスへマッピング

---

## Not Found：404

```typescript
it('returns 404 when app does not exist', async () => {
  const res = await ctx.controller.get('00000000-0000-0000-0000-000000000000');
  expect(res.status).toBe(404);
  expect(res.body.success).toBe(false);
  expect(res.body.error?.code).toBe('NOT_FOUND');
});

it('delete on non-existent app returns 404', async () => {
  const res = await ctx.controller.delete('unknown-id');
  expect(res.status).toBe(404);
});
```

**404 Not Found の条件：**

- `AppError('NOT_FOUND', ...)` がスロー
- `findActiveById()` が null を返す
- soft-deleted レコードも「見えない」として 404

---

## DTO 変換と削除フィールド隠蔽

### presentApp / presentTodo の設計

```typescript
export function presentApp(app: AppEntity) {
  return {
    id: app.id,
    name: app.name,
    createdAt: app.createdAt,
    updatedAt: app.updatedAt,
    // ❌ deletedAt は含めない
  };
}

export function presentTodo(todo: TodoEntity) {
  return {
    id: todo.id,
    appId: todo.appId,
    title: todo.title,
    completed: todo.completed,
    createdAt: todo.createdAt,
    updatedAt: todo.updatedAt,
    // ❌ deletedAt は含めない
  };
}
```

**設計上のポイント：**

1. **内部フィールドの隠蔽** — `deletedAt` は repository / service で使うが、API では返さない
2. **スキーマ分離** — エンティティ（内部用）と DTO（外部用）の shape が異なる
3. **型安全性** — TypeScript の型システムで自動的に確認可能

### DTO にフィールドが含まれていないことのテスト

```typescript
it('list returns app DTOs without deletedAt', async () => {
  await ctx.controller.create({ name: 'App 1' });
  await ctx.controller.create({ name: 'App 2' });
  
  const res = await ctx.controller.list();
  expect(res.status).toBe(200);
  
  const apps = res.body.data as Record<string, unknown>[];
  apps.forEach(app => {
    expect(app).not.toHaveProperty('deletedAt');
  });
});

it('get returns app DTO without deletedAt', async () => {
  const createRes = await ctx.controller.create({ name: 'Test' });
  const appId = (createRes.body.data as Record<string, unknown>).id as string;
  
  const getRes = await ctx.controller.get(appId);
  expect((getRes.body.data as Record<string, unknown>)).not.toHaveProperty('deletedAt');
});
```

---

## Update 操作での複合エラー

```typescript
describe('update', () => {
  it('returns 404 when app does not exist', async () => {
    const res = await ctx.controller.update('unknown', { name: 'New' });
    expect(res.status).toBe(404);
  });

  it('returns 422 when update payload is invalid', async () => {
    const created = await ctx.controller.create({ name: 'Original' });
    const appId = (created.body.data as Record<string, unknown>).id as string;
    
    const res = await ctx.controller.update(appId, { name: '   ' });
    expect(res.status).toBe(422);
  });

  it('returns 409 when new name conflicts with another app', async () => {
    await ctx.controller.create({ name: 'App 1' });
    const created = await ctx.controller.create({ name: 'App 2' });
    const appId = (created.body.data as Record<string, unknown>).id as string;
    
    const res = await ctx.controller.update(appId, { name: 'App 1' });
    expect(res.status).toBe(409);
  });

  it('returns 200 on successful update', async () => {
    const created = await ctx.controller.create({ name: 'Original' });
    const appId = (created.body.data as Record<string, unknown>).id as string;
    
    const res = await ctx.controller.update(appId, { name: 'Updated' });
    expect(res.status).toBe(200);
    expect(res.body.success).toBe(true);
    expect((res.body.data as Record<string, unknown>).name).toBe('Updated');
  });
});
```

---

## エラーレスポンスの統一

すべてのエラーレスポンスが同じ shape を持つことを検証：

```typescript
it('error responses always include error.code and error.message', async () => {
  const responses = [
    await ctx.controller.create({}), // 422
    await ctx.controller.get('unknown'), // 404
  ];

  responses.forEach(res => {
    if (!res.body.success) {
      expect(res.body.error).toBeDefined();
      expect(res.body.error?.code).toBeDefined();
      expect(res.body.error?.message).toBeDefined();
    }
  });
});
```

---

## controllers テストの成果

controllers 層統合テストにより：

1. **ステータスコードマッピングが検証される** — 201 / 200 / 422 / 409 / 404 の使い分けが正確
2. **DTO shape が保証される** — 内部フィールド（deletedAt）が漏れない
3. **エラーハンドリングが統一される** — すべてのエラーが同じ shape で返却
4. **バリデーションエラーが分類される** — 入力エラー（422）と business logic エラー（409 / 404）が区別される

---

## まとめ

controllers 層テストは、「HTTP インターフェースが正しく実装されている」ことを唯一証明する層である。

特に **DTO 変換**（deletedAt 隠蔽）や **ステータスコード選定**は、API ドキュメント（OpenAPI）と一貫性を保つために必須である。

controllers テストなしでは、以下のようなバグが検出されない：

- 作成時に 200 を返しているが 201 が必要
- エラーレスポンスが統一されていない
- 内部フィールドが JSON に漏れている

controllers テストは、API の「公約」を実装レベルで保証する重要な層である。
