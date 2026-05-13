# HTTP プレゼンター層：deletedAt を隠蔽し API スキーマを保護する

## 対象読者

- Clean Architecture で DTO / presenter 層を実装している
- API レスポンスから内部フィールドを隠蔽したい
- スキーマ分離（internal entity vs API DTO）の設計を理解したい

## この記事が扱う範囲

- `AppEntity` vs `AppDTO` のスキーマ分離
- `presentApp()` / `presentTodo()` による変換
- `deletedAt` の隠蔽メカニズム
- DTO の shape テスト
- timestamp フォーマットの検証

---

## 背景：API スキーマと内部エンティティの分離

Clean Architecture では、**internal entity** と **API DTO** が異なる shape を持つ：

```typescript
// Internal Entity - ソフトデリート用に deletedAt を持つ
export type AppEntity = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null; // ← 内部用
};

// API DTO - deletedAt は含めない
type AppDTO = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
};
```

理由：

1. **実装の詳細** — ソフトデリート戦略は API クライアントに無関係
2. **スキーマ安定性** — 内部実装が変わってもAPI形状は変わらない
3. **セキュリティ** — 削除されたリソースの「削除時刻」が外部に漏れない

---

## HttpPresenter の実装

### presentApp の設計

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
```

**選択したフィールド：**

| フィールド | 理由 |
|---|---|
| `id` | リソース識別子 |
| `name` | ビジネスデータ |
| `createdAt` | 監査ログ用途 |
| `updatedAt` | 監査ログ用途 |
| ~~`deletedAt`~~ | 内部実装詳細・隠蔽 |

### presentTodo の設計

```typescript
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

### presentSuccess / presentError

```typescript
export type JsonHttpResponse = {
  status: number;
  body: ApiResponseBody;
};

export type ApiResponseBody = {
  data: unknown;
  success: boolean;
  error?: ErrorBody;
};

export function presentSuccess(data: unknown, status = 200): JsonHttpResponse {
  return {
    status,
    body: { data, success: true },
  };
}

export function presentError(code: string, message: string, status = 400): JsonHttpResponse {
  return {
    status,
    body: {
      data: null,
      success: false,
      error: { code, message },
    },
  };
}
```

---

## deletedAt 隠蔽テスト

### DTO shape 検証

```typescript
describe('HttpPresenter integration', () => {
  describe('presentApp', () => {
    it('returns the correct DTO shape from a real app entity', async () => {
      const { appInteractor } = setup();
      const app = await appInteractor.create({ name: 'Presenter Test' });
      const dto = presentApp(app);
      
      expect(dto).toEqual({
        id: app.id,
        name: app.name,
        createdAt: app.createdAt,
        updatedAt: app.updatedAt,
      });
    });

    it('DTO does not include deletedAt', async () => {
      const { appInteractor } = setup();
      const app = await appInteractor.create({ name: 'No Deleted' });
      const dto = presentApp(app) as Record<string, unknown>;
      
      expect(dto).not.toHaveProperty('deletedAt');
    });
  });
});
```

**テスト設計のポイント：**

1. **toEqual()** — DTO が期待する正確な shape を持つ
2. **not.toHaveProperty('deletedAt')** — DTO に余分なフィールドがない
3. **as Record<string, unknown>** — Object key existence check を型安全に

### timestamp フォーマット検証

```typescript
it('createdAt and updatedAt are ISO strings from the real entity', async () => {
  const { appInteractor } = setup();
  const app = await appInteractor.create({ name: 'Timestamps' });
  const dto = presentApp(app);
  
  expect(dto.createdAt).toMatch(/^\d{4}-\d{2}-\d{2}T/);
  expect(dto.updatedAt).toMatch(/^\d{4}-\d{2}-\d{2}T/);
});
```

**ISO 8601 形式の確認：** `2024-01-01T00:00:00.000Z`

---

## presentTodo のテスト

### TodoDTO の shape

```typescript
describe('presentTodo', () => {
  it('returns the correct DTO shape from a real todo entity', async () => {
    const { appInteractor, todoInteractor } = setup();
    const app = await appInteractor.create({ name: 'App For Todo' });
    const todo = await todoInteractor.create({ appId: app.id, title: 'My Todo' });
    const dto = presentTodo(todo);
    
    expect(dto).toEqual({
      id: todo.id,
      appId: todo.appId,
      title: todo.title,
      completed: todo.completed,
      createdAt: todo.createdAt,
      updatedAt: todo.updatedAt,
    });
  });

  it('does not expose deletedAt in the DTO', async () => {
    const { appInteractor, todoInteractor } = setup();
    const app = await appInteractor.create({ name: 'App' });
    const todo = await todoInteractor.create({ appId: app.id, title: 'Todo' });
    const dto = presentTodo(todo) as Record<string, unknown>;
    
    expect(dto).not.toHaveProperty('deletedAt');
  });

  it('preserves the completed flag correctly', async () => {
    const { appInteractor, todoInteractor } = setup();
    const app = await appInteractor.create({ name: 'App' });
    
    const todo1 = await todoInteractor.create({ appId: app.id, title: 'Fresh' });
    expect(presentTodo(todo1).completed).toBe(false);
    
    const updated = await todoInteractor.update({
      appId: app.id,
      todoId: todo1.id,
      completed: true,
    });
    expect(presentTodo(updated).completed).toBe(true);
  });
});
```

---

## presentSuccess / presentError の検証

### 成功レスポンスのスキーマ

```typescript
describe('presentSuccess', () => {
  it('creates a success response with default status 200', () => {
    const response = presentSuccess({ id: '1', name: 'Test' });
    expect(response.status).toBe(200);
    expect(response.body.success).toBe(true);
    expect(response.body.data).toEqual({ id: '1', name: 'Test' });
    expect(response.body.error).toBeUndefined();
  });

  it('creates a success response with custom status', () => {
    const response = presentSuccess({ id: '1' }, 201);
    expect(response.status).toBe(201);
    expect(response.body.success).toBe(true);
  });

  it('wraps DTO in data field', () => {
    const dto = { id: '1', name: 'App' };
    const response = presentSuccess(dto);
    expect(response.body.data).toBe(dto);
  });
});
```

### エラーレスポンスのスキーマ

```typescript
describe('presentError', () => {
  it('creates an error response with default status', () => {
    const response = presentError('NOT_FOUND', 'App not found');
    expect(response.status).toBe(400);
    expect(response.body.success).toBe(false);
    expect(response.body.error?.code).toBe('NOT_FOUND');
    expect(response.body.error?.message).toBe('App not found');
    expect(response.body.data).toBeNull();
  });

  it('uses custom status when provided', () => {
    const response = presentError('NOT_FOUND', 'Not found', 404);
    expect(response.status).toBe(404);
  });

  it('error response does not include success=true', () => {
    const response = presentError('CONFLICT', 'Duplicate');
    expect(response.body.success).toBe(false);
  });
});
```

---

## スキーマ分離の効果

### 利点1：API の安定性

Entity に新しいフィールドを追加しても、API shape は変わらない：

```typescript
// ← 将来：MySQLへの移行時に timestamp_created_utc 等を追加
export type AppEntity = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null;
  // timestamp_created_utc?: number;  // 新フィールド
};

// API DTO は変わらない
type AppDTO = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
};
```

### 利点2：セキュリティ

削除済みリソースについて、削除タイムスタンプは外部に漏れない：

```typescript
// 内部で削除済みリソースを特定
const deleted = await repo.findAll(); // deletedAt !== null を持つ
// しかし API には削除済みの deletedAt は返さない
```

---

## プレゼンター層テストの成果

HttpPresenter テストにより：

1. **API DTO が正確に定義される** — 期待する shape を強制
2. **deletedAt 隠蔽が検証される** — 内部フィールドが漏れない
3. **ステータスコード対応** — success / error レスポンスが統一
4. **スキーマドリブンテスト** — OpenAPI / Swagger との自動生成に対応可能

---

## まとめ

HTTP プレゼンター層は、「internal entity」と「API DTO」を明確に分離し、実装の詳細をクライアントから隠蔽する責務を持つ。

特に重要な工夫：

1. **deletedAt 隠蔽** — ソフトデリート戦略が API に透明
2. **スキーマ分離** — 内部の進化が API に影響しない
3. **response shape の統一** — success / error が同じ format

プレゼンター層をテストで検証することで、「API が正しく定義されている」という確信を得られ、チーム全体の開発品質が向上する。
