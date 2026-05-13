# Shared Storageパターンで複数リポジトリのデータ一貫性を保つ

## 対象読者

- 複数のリポジトリ（AppRepository / TodoRepository）が存在する設計をしている
- リポジトリ間で cascade delete を実装したい
- in-memory storage の「共有」と「隔離」の設計を理解したい

## この記事が扱う範囲

- `InMemoryStorage` の設計
- AppRepository と TodoRepository による storage 共有メカニズム
- Shared Storage を通じたカスケード削除
- ストレージ隔離（複数インスタンスで互いに影響しない）
- `storage.clear()` による全体リセット

---

## 背景：複数リポジトリでのデータ一貫性が難しい

マルチリポジトリ設計では以下の課題が生じる：

- **App 削除時に Todo も削除する** — cascade delete ロジック
- **リポジトリ間の参照** — AppRepository の削除を TodoRepository が認識
- **テストの隔離** — テスト間でデータが混在しない

これらを解決する設計が **Shared Storage パターン**である。

---

## InMemoryStorage の設計

```typescript
export type InMemoryStorage = {
  apps: Map<string, AppEntity>;
  todos: Map<string, TodoEntity>;
  clear(): void;
};

export function createInMemoryStorage(): InMemoryStorage {
  const apps = new Map<string, AppEntity>();
  const todos = new Map<string, TodoEntity>();

  return {
    apps,
    todos,
    clear() {
      apps.clear();
      todos.clear();
    },
  };
}
```

**設計上のポイント：**

1. **公開 API** — `apps` / `todos` 両方を外部に公開（リポジトリ実装用）
2. **`clear()` メソッド** — 両マップをアトミックにクリア → テスト隔離
3. **型安全性** — エンティティ型が明示される

---

## リポジトリ実装：共有 storage の活用

### AppRepository 実装

```typescript
export function createInMemoryAppRepository(storage: InMemoryStorage): AppRepository {
  async function save(app: AppEntity): Promise<void> {
    storage.apps.set(app.id, { ...app }); // 防御的コピーで保存
  }

  async function findActiveById(id: string): Promise<AppEntity | null> {
    const app = storage.apps.get(id);
    if (!app || app.deletedAt !== null) return null;
    return { ...app }; // 防御的コピーで返却
  }

  async function listActive(): Promise<AppEntity[]> {
    return Array.from(storage.apps.values())
      .filter(app => app.deletedAt === null)
      .map(app => ({ ...app })); // 各要素を防御的コピー
  }

  async function existsActiveByName(name: string, excludeId?: string): Promise<boolean> {
    return Array.from(storage.apps.values()).some(app =>
      app.deletedAt === null &&
      app.name === name &&
      (!excludeId || app.id !== excludeId)
    );
  }

  return { save, findActiveById, listActive, existsActiveByName };
}
```

### TodoRepository 実装

```typescript
export function createInMemoryTodoRepository(storage: InMemoryStorage): TodoRepository {
  async function listActiveByAppId(appId: string): Promise<TodoEntity[]> {
    return Array.from(storage.todos.values())
      .filter(todo => todo.appId === appId && todo.deletedAt === null)
      .map(todo => ({ ...todo }));
  }

  async function findActiveByIdAndAppId(id: string, appId: string): Promise<TodoEntity | null> {
    const todo = storage.todos.get(id);
    if (!todo || todo.appId !== appId || todo.deletedAt !== null) return null;
    return { ...todo };
  }

  // ... その他のメソッド
}
```

**共有 storage が実現する責任分離：**

1. **AppRepository** — storage の `apps` Map を操作
2. **TodoRepository** — storage の `todos` Map を操作
3. **Shared storage** — 両リポジトリが「同じ storage インスタンス」を参照 → データが一貫

---

## Cascade Delete：shared storage を通じた実装

### AppInteractor での cascade delete

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

**cascade delete の流れ：**

1. App を soft-delete → `appRepository.save()` で storage.apps を更新
2. App の id でスコープされた Todo を取得 → `todoRepository.listActiveByAppId()`
3. 各 Todo を soft-delete → `todoRepository.save()` で storage.todos を更新

重要なのは、**両リポジトリが同じ storage インスタンスを参照している**ため、App 削除が Todo の検索に即座に反映されることである。

---

## Shared Storage テスト

### 基本的な共有の検証

```typescript
describe('InMemoryRepositories cross-repo integration', () => {
  describe('shared storage between app and todo repos', () => {
    it('save in appRepo and retrieve from the same storage', async () => {
      const storage = createInMemoryStorage();
      const appRepo = createInMemoryAppRepository(storage);
      const app = makeApp('a1', 'App');
      await appRepo.save(app);
      expect(storage.apps.size).toBe(1);
      expect(storage.apps.get('a1')?.name).toBe('App');
    });

    it('both repos share the same storage instance', async () => {
      const storage = createInMemoryStorage();
      const appRepo = createInMemoryAppRepository(storage);
      const todoRepo = createInMemoryTodoRepository(storage);
      
      const app = makeApp('a1', 'App');
      await appRepo.save(app);
      
      const todo = makeTodo('t1', 'a1', 'Todo');
      await todoRepo.save(todo);
      
      // 同じ storage で両データが見える
      expect(storage.apps.size).toBe(1);
      expect(storage.todos.size).toBe(1);
    });
  });
});
```

### storage.clear() による全体リセット

```typescript
it('storage.clear() resets both repos at once', async () => {
  const storage = createInMemoryStorage();
  const appRepo = createInMemoryAppRepository(storage);
  const todoRepo = createInMemoryTodoRepository(storage);
  
  await appRepo.save(makeApp('a1', 'App'));
  await todoRepo.save(makeTodo('t1', 'a1', 'Todo'));
  
  // クリア前：両方データがある
  expect(await appRepo.listActive()).toHaveLength(1);
  expect(await todoRepo.listActiveByAppId('a1')).toHaveLength(1);
  
  // クリア
  storage.clear();
  
  // クリア後：両方空っぽ
  expect(await appRepo.listActive()).toHaveLength(0);
  expect(await todoRepo.listActiveByAppId('a1')).toHaveLength(0);
});
```

**重要なポイント：**

- `storage.clear()` は **両 Map をアトミックにクリア** → テスト間の隔離が確実
- beforeEach で `storage.clear()` を呼ぶだけで、すべてのリポジトリがリセット

---

## 防御的コピーとの組み合わせ

Shared Storage パターンでは、**防御的コピー**が極めて重要である。

```typescript
// ❌ 悪い例：直接参照を返す
async function findActiveById(id: string): Promise<AppEntity | null> {
  return storage.apps.get(id); // 呼び出し側が改変すると storage が汚染
}

// ✅ 良い例：防御的コピーで返す
async function findActiveById(id: string): Promise<AppEntity | null> {
  const app = storage.apps.get(id);
  if (!app || app.deletedAt !== null) return null;
  return { ...app }; // スプレッド演算子で shallow copy
}
```

テスト：

```typescript
it('returned entity is a defensive copy, not a reference', async () => {
  const storage = createInMemoryStorage();
  const appRepo = createInMemoryAppRepository(storage);
  
  const app = makeApp('a1', 'Original');
  await appRepo.save(app);
  
  const found = await appRepo.findActiveById('a1');
  expect(found).not.toBe(app); // 異なるオブジェクト
  
  // caller が改変しても storage は影響されない
  if (found) {
    (found as any).name = 'HACKED';
  }
  
  const refetched = await appRepo.findActiveById('a1');
  expect(refetched?.name).toBe('Original'); // 変わっていない
});
```

---

## ストレージ隔離：複数インスタンスの独立性

```typescript
describe('separate storage instances are isolated', () => {
  it('two separate storages do not share data', async () => {
    const storage1 = createInMemoryStorage();
    const storage2 = createInMemoryStorage();
    
    const repo1 = createInMemoryAppRepository(storage1);
    const repo2 = createInMemoryAppRepository(storage2);
    
    // storage1 に save
    await repo1.save(makeApp('a1', 'App 1'));
    
    // storage2 には見えない
    expect(await repo2.listActive()).toHaveLength(0);
  });
});
```

**重要なポイント：**

- storage を創成する度に、独立したインスタンスが生成される
- 異なる storage インスタンスは互いに影響しない
- テストで複数の独立したデータセットが必要な場合に活用

---

## Shared Storage テストの成果

Shared Storage パターン検証により：

1. **複数リポジトリの一貫性が保証される** — 同じ storage で両リポジトリが動作
2. **Cascade delete が機能する** — App 削除が Todo の検索に即座に反映
3. **防御的コピーが有効** — caller の改変が storage を汚さない
4. **テスト隔離が確実** — storage.clear() で全体を一度にリセット
5. **ストレージ独立性** — 複数の独立したデータセットが並行可能

---

## まとめ

Shared Storage パターンは、複数リポジトリが「同じデータソース」を参照することで、**参照整合性と cascade delete** を同時に実現する設計である。

テストにおいて重要な工夫は：

1. **storage を明示的に作成・注入** — DI で storage を共有
2. **防御的コピーを厳格に** — 返却値の改変が storage に影響しない
3. **clear() メソッド** — テスト間の隔離を確実に

この設計により、マルチリポジトリ構成でも複合条件の検証や cascade delete の正確性が保証される。
