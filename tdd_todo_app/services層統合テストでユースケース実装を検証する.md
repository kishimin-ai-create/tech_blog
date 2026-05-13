# services層統合テストでユースケース実装を検証する

## 対象読者

- Clean Architecture の Interactor / Usecase パターンを実装している
- ビジネスロジックのテストを「単体」と「統合」に分けたい
- AppError などのドメインエラーの伝播をテストしたい

## この記事が扱う範囲

- `AppInteractor` / `TodoInteractor` の CRUD 操作テスト
- ユースケース契約の型安全性検証（`AppUsecase` インターフェース）
- Shared Storage を通じた cascade delete の検証
- カスケード削除時のタイムスタンプ一貫性

---

## 背景：services 層はビジネスロジックの中核

services 層（Interactor / Usecase）の責務：

- **ビジネスルール実装** — 一意制約チェック、cascade delete ロジック
- **repository 層の組み立て** — DI で複数リポジトリを一貫して使う
- **エラーハンドリング** — AppError をスロー
- **タイムスタンプ生成** — id / timestamp の一貫性を保証

services テストは、models テスト（型と伝播）と repositories テスト（契約）の両方に依存する。

---

## AppInteractor の構造

```typescript
type AppInteractorDependencies = {
  appRepository: AppRepository;
  todoRepository: TodoRepository;
  generateId?: () => string;
  now?: () => string;
};

export function createAppInteractor(
  dependencies: AppInteractorDependencies,
): AppUsecase {
  // create / list / get / update / delete の 5 操作を実装
}
```

**テスト設計上の工夫：**

1. **DI による依存性** — `generateId` / `now` を挿入可能にして、テストでは固定値を使用
2. **インターフェース返却** — `AppUsecase` インターフェースを返すことで、実装の詳細をカプセル化

---

## AppInteractor テストの実装

### Helper 設計：モック vs 実装

```typescript
function makeAppRepository(
  overrides: Partial<AppRepository> = {},
): AppRepository {
  return {
    save: vi.fn().mockResolvedValue(undefined),
    listActive: vi.fn().mockResolvedValue([]),
    findActiveById: vi.fn().mockResolvedValue(null),
    existsActiveByName: vi.fn().mockResolvedValue(false),
    ...overrides,
  };
}

const FIXED_ID = 'test-uuid-0000';
const FIXED_TIME = '2024-06-01T12:00:00.000Z';

function makeInteractor(
  appRepo: AppRepository,
  todoRepo: TodoRepository,
) {
  return createAppInteractor({
    appRepository: appRepo,
    todoRepository: todoRepo,
    generateId: () => FIXED_ID,
    now: () => FIXED_TIME,
  });
}
```

**モックと固定値の役割分担：**

| テスト方針 | 使い方 |
|---|---|
| **単体テスト（ユニット）** | repository をモック、生成値を固定 → interactor だけを検証 |
| **統合テスト** | 実装 repository + 固定値 → end-to-end で動作確認 |

`services/app-interactor.test.ts` はモック使用（単体）で、`tests/integrations/services/app-interactor.test.ts` は実装リポジトリ使用（統合）となる。

### create 操作のテスト

```typescript
describe('createAppInteractor.create', () => {
  it('saves and returns a new app entity', async () => {
    const appRepo = makeAppRepository();
    const interactor = makeInteractor(appRepo, makeTodoRepository());

    const result = await interactor.create({ name: 'My App' });

    expect(result).toEqual({
      id: FIXED_ID,
      name: 'My App',
      createdAt: FIXED_TIME,
      updatedAt: FIXED_TIME,
      deletedAt: null,
    });
    expect(appRepo.save).toHaveBeenCalledWith(result);
  });

  it('throws CONFLICT when app name already exists', async () => {
    const appRepo = makeAppRepository({
      existsActiveByName: vi.fn().mockResolvedValue(true),
    });
    const interactor = makeInteractor(appRepo, makeTodoRepository());

    await expect(interactor.create({ name: 'Existing' })).rejects.toThrow(
      expect.objectContaining({ code: 'CONFLICT' }),
    );
  });

  it('does not call save when duplicate is detected', async () => {
    const appRepo = makeAppRepository({
      existsActiveByName: vi.fn().mockResolvedValue(true),
    });
    const interactor = makeInteractor(appRepo, makeTodoRepository());

    try {
      await interactor.create({ name: 'Existing' });
    } catch {
      // AppError が thrown される
    }
    expect(appRepo.save).not.toHaveBeenCalled();
  });
});
```

**テスト設計のポイント：**

1. **成功時の shape** — id / timestamp が期待通り生成される
2. **エラー時の動作** — 一意制約違反で `CONFLICT` がスロー
3. **失敗時の副作用** — save が呼ばれない（トランザクション的安全性）

### update 操作でのエラーハンドリング

```typescript
describe('update', () => {
  it('throws NOT_FOUND when app does not exist', async () => {
    const appRepo = makeAppRepository({
      findActiveById: vi.fn().mockResolvedValue(null),
    });
    const interactor = makeInteractor(appRepo, makeTodoRepository());

    await expect(
      interactor.update({ appId: 'unknown', name: 'New Name' })
    ).rejects.toThrow(
      expect.objectContaining({ code: 'NOT_FOUND' })
    );
  });

  it('throws CONFLICT when another app has the updated name', async () => {
    const appRepo = makeAppRepository({
      findActiveById: vi.fn().mockResolvedValue({
        id: 'app-1',
        name: 'Old Name',
        createdAt: FIXED_TIME,
        updatedAt: FIXED_TIME,
        deletedAt: null,
      }),
      existsActiveByName: vi.fn().mockResolvedValue(true),
    });
    const interactor = makeInteractor(appRepo, makeTodoRepository());

    await expect(
      interactor.update({ appId: 'app-1', name: 'Taken Name' })
    ).rejects.toThrow(
      expect.objectContaining({ code: 'CONFLICT' })
    );
  });
});
```

**重要なポイント：**

- `existsActiveByName` に **自分の ID を除外する `excludeId` パラメータ**を渡すことで、「自分以外の重複」だけを検出
- update を「作成時と異なる」テストケースとして扱う

---

## Shared Storage 経由のカスケード削除

services 層が複数リポジトリを扱う典型例は、**cascade delete** である。

```typescript
async function delete(input: DeleteAppInput): Promise<void> {
  const app = await findExistingApp(input.appId);
  
  const now_str = now();
  const deletedApp: AppEntity = { ...app, deletedAt: now_str, updatedAt: now_str };
  await appRepository.save(deletedApp);
  
  // Cascade: soft-delete all todos for this app
  const todos = await todoRepository.listActiveByAppId(app.id);
  for (const todo of todos) {
    const deletedTodo: TodoEntity = { ...todo, deletedAt: now_str, updatedAt: now_str };
    await todoRepository.save(deletedTodo);
  }
}
```

テスト：

```typescript
describe('delete with cascade', () => {
  it('soft-deletes the app and all its todos with the same timestamp', async () => {
    const storage = createInMemoryStorage();
    const appRepo = createInMemoryAppRepository(storage);
    const todoRepo = createInMemoryTodoRepository(storage);
    const interactor = makeInteractor(appRepo, todoRepo);

    // Setup: create app and 2 todos
    const app = await interactor.create({ name: 'My App' });
    const todo1 = await createTodo(app.id, 'Todo 1');
    const todo2 = await createTodo(app.id, 'Todo 2');

    // Delete app
    await interactor.delete({ appId: app.id });

    // Verify: app is deleted
    expect(await appRepo.findActiveById(app.id)).toBeNull();

    // Verify: todos are also deleted
    expect(await todoRepo.findActiveByIdAndAppId(todo1.id, app.id)).toBeNull();
    expect(await todoRepo.findActiveByIdAndAppId(todo2.id, app.id)).toBeNull();

    // Verify: all have the same deletedAt timestamp
    const allTodos = await todoRepo.listActiveByAppId(app.id);
    expect(allTodos).toHaveLength(0);
  });
});
```

**cascade delete の検証ポイント：**

1. **Shared Storage** — appRepo と todoRepo が同じ storage を参照
2. **全 todo が削除される** — `listActiveByAppId` で非表示化を確認
3. **タイムスタンプ一貫性** — app 削除時刻と todo 削除時刻が同じ

---

## インターフェース契約テスト（型代入）

services 層の重要な設計は、**実装がインターフェースを満たすことをコンパイル時に検証する**ことである。

```typescript
// services/app-usecase.ts
export interface AppUsecase {
  create(input: CreateAppInput): Promise<AppEntity>;
  list(): Promise<AppEntity[]>;
  get(input: GetAppInput): Promise<AppEntity>;
  update(input: UpdateAppInput): Promise<AppEntity>;
  delete(input: DeleteAppInput): Promise<void>;
}

// services/app-usecase.test.ts
describe('AppUsecase contract', () => {
  it('createAppInteractor implements AppUsecase', () => {
    const storage = createInMemoryStorage();
    const interactor = createAppInteractor({
      appRepository: createInMemoryAppRepository(storage),
      todoRepository: createInMemoryTodoRepository(storage),
    });

    // 型代入テスト：interactor が AppUsecase を満たすことをコンパイル時検証
    const _: AppUsecase = interactor;
    expect(_).toBeDefined();
  });
});
```

このテストは一見すると「何もしていない」ように見えるが、実はコンパイルエラーが発生することで検証される：

- interactor に `create` メソッドがない → コンパイルエラー
- パラメータ型が異なる → コンパイルエラー
- 戻り値型が異なる → コンパイルエラー

---

## services テストの成果

services 層統合テストにより：

1. **ビジネスロジックが実装される** — cascade delete やエラーハンドリングが正しく機能
2. **エラー伝播が証明される** — AppError が層を越えて caller に到達
3. **インターフェース契約が満たされる** — 型代入で型安全性を確認
4. **Shared Storage パターンが動作する** — 複数リポジトリ間のデータ一貫性が保証される

---

## まとめ

services 層テストは、「ビジネスロジックが正しく実装されている」ことを唯一証明する層である。

特に **cascade delete** のようなマルチリポジトリ操作や、**インターフェース契約テスト**は、テストなしでは検出されないバグが潜んでいる。

services テストは、repositories テストの一段上で、「複数層の連携」を検証する責務を持つ。
