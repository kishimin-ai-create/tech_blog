# in-memory repository の防御的コピーテスト：クローン不変性を検証する

## 対象読者

- in-memory repository を実装している
- 返却値の改変がストアを汚さないことを確認したい
- 防御的コピーの「意図」と「検証方法」を理解したい

## この記事が扱う範囲

- 防御的コピー（defensive copy）の設計
- クローン不変性テスト（clone immutability）
- shallow copy vs deep copy
- スプレッド演算子による防御

---

## 背景：参照型の危険性

JavaScript / TypeScript ではオブジェクトは参照型である。

```typescript
// ❌ 危険な例
const original = { id: 'a1', name: 'Original' };
const returned = original; // 参照をそのまま返す

returned.name = 'Mutated'; // caller が改変
console.log(original.name); // 'Mutated' - ストアが汚染された！
```

in-memory repository がストアから直接参照を返していたら、caller がそれを改変するたびにストアが破壊される。

---

## 防御的コピーの実装

### 保存時のコピー

```typescript
export function createInMemoryAppRepository(storage: InMemoryStorage): AppRepository {
  async function save(app: AppEntity): Promise<void> {
    // ❌ 悪い例：直接保存
    // storage.apps.set(app.id, app);

    // ✅ 良い例：防御的コピーで保存
    storage.apps.set(app.id, { ...app });
  }

  // ...
}
```

### 返却時のコピー

```typescript
async function findActiveById(id: string): Promise<AppEntity | null> {
  const app = storage.apps.get(id);
  if (!app || app.deletedAt !== null) return null;
  
  // ❌ 悪い例：直接返す
  // return app;

  // ✅ 良い例：防御的コピーで返す
  return { ...app };
}

async function listActive(): Promise<AppEntity[]> {
  return Array.from(storage.apps.values())
    .filter(app => app.deletedAt === null)
    .map(app => ({ ...app })); // 各要素を防御的コピー
}
```

---

## 防御的コピーテスト

### 基本的な不変性テスト

```typescript
describe('repository returns clones (no mutation through returned reference)', () => {
  it('mutating a returned app entity does not affect stored data', async () => {
    const storage = createInMemoryStorage();
    const appRepo = createInMemoryAppRepository(storage);
    
    // 保存
    await appRepo.save(makeApp('a1', 'Original'));
    
    // 取得
    const found = await appRepo.findActiveById('a1');
    
    // caller が改変
    if (found) found.name = 'Mutated';
    
    // 再取得 - ストアが汚れていないことを確認
    const found2 = await appRepo.findActiveById('a1');
    expect(found2?.name).toBe('Original'); // 変わってない
  });

  it('mutating a returned todo entity does not affect stored data', async () => {
    const storage = createInMemoryStorage();
    const todoRepo = createInMemoryTodoRepository(storage);
    
    await todoRepo.save(makeTodo('t1', 'a1', 'Original Title'));
    const found = await todoRepo.findActiveById('a1', 't1');
    
    if (found) found.title = 'Mutated';
    
    const found2 = await todoRepo.findActiveById('a1', 't1');
    expect(found2?.title).toBe('Original Title');
  });
});
```

**テスト設計のポイント：**

1. **意図的な改変** — `found.name = 'Mutated'` で caller が改変を試みる
2. **再取得で確認** — 改変後に再度データを取得して、ストアが変わってないことを検証
3. **複数フィールド** — 異なる型のフィールドで防御を確認

### listActive の防御テスト

```typescript
it('mutating an element in returned list does not affect stored data', async () => {
  const storage = createInMemoryStorage();
  const appRepo = createInMemoryAppRepository(storage);
  
  await appRepo.save(makeApp('a1', 'App 1'));
  await appRepo.save(makeApp('a2', 'App 2'));
  
  // リスト取得
  const list = await appRepo.listActive();
  expect(list).toHaveLength(2);
  
  // list 内の要素を改変
  if (list[0]) list[0].name = 'HACKED';
  
  // 再取得で確認
  const list2 = await appRepo.listActive();
  expect(list2[0].name).toBe('App 1'); // 変わってない
});
```

---

## Shallow Copy vs Deep Copy

### shallow copy の十分性

```typescript
export type AppEntity = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null;
};
```

AppEntity はすべてプリミティブ型（string / null）なため、**shallow copy（スプレッド演算子）で十分**である。

```typescript
const original = { id: 'a1', name: 'Original' };
const copy = { ...original }; // shallow copy

copy.name = 'Mutated';
console.log(original.name); // 'Original' - 保護される
```

### 深いネストがある場合

もし AppEntity に配列や nested object があったら、deep copy が必要になる。

```typescript
// ❌ shallow copy では不十分な例
export type AppEntity = {
  id: string;
  name: string;
  tags: string[]; // 配列はオブジェクト参照
};

const original = { id: 'a1', name: 'Original', tags: ['important'] };
const copy = { ...original };

copy.tags.push('hacked'); // shallow copy では tags 配列は共有
console.log(original.tags); // ['important', 'hacked'] - 汚染！
```

このプロジェクトでは AppEntity と TodoEntity が flat な構造なため、**shallow copy で十分**。

---

## 防御的コピーのコスト

防御的コピーには **パフォーマンスコスト**がある：

```typescript
// ❌ 毎回コピーするのは高コスト
async function findActiveById(id: string): Promise<AppEntity | null> {
  const app = storage.apps.get(id);
  if (!app || app.deletedAt !== null) return null;
  return { ...app }; // コピー
}

async function listActive(): Promise<AppEntity[]> {
  return Array.from(storage.apps.values())
    .filter(app => app.deletedAt === null)
    .map(app => ({ ...app })); // N 個のコピー
}
```

**実装上の判断：**

- **in-memory 実装** → コストは無視できる（メモリ内操作）
- **本物の DB 実装** → コピーは不要（DB 接続は返却オブジェクトが新規で返される）
- **キャッシュ層** → コピーがなければ caller の改変でキャッシュが破壊される

このプロジェクトでは、将来 MySQL 実装に移行することを想定しているため、**in-memory でもコピーを徹底**して、実装習慣を統一している。

---

## 複合条件での防御テスト

### soft-delete フィルタリングとコピーの組み合わせ

```typescript
it('soft-deleted entities are filtered and safe from mutation', async () => {
  const storage = createInMemoryStorage();
  const appRepo = createInMemoryAppRepository(storage);
  
  // アクティブなアプリ
  await appRepo.save(makeApp('a1', 'Active'));
  
  // ソフトデリート済みアプリ
  await appRepo.save(makeApp('a2', 'Deleted', '2024-01-02T00:00:00.000Z'));
  
  // listActive は a1 だけを返す
  const list = await appRepo.listActive();
  expect(list).toHaveLength(1);
  expect(list[0].id).toBe('a1');
  
  // caller が改変
  list[0].name = 'Hacked';
  
  // 再取得で確認
  const list2 = await appRepo.listActive();
  expect(list2[0].name).toBe('Active');
});
```

---

## repository 別の防御テスト

### AppRepository のテスト

```typescript
it('existsActiveByName does not leak internal state', async () => {
  const storage = createInMemoryStorage();
  const appRepo = createInMemoryAppRepository(storage);
  
  await appRepo.save(makeApp('a1', 'Important'));
  
  // boolean のみ返る（object reference なし）
  const exists = await appRepo.existsActiveByName('Important');
  expect(exists).toBe(true);
  
  // 返り値が boolean なので改変不可能
});
```

### TodoRepository のテスト

```typescript
it('findActiveByIdAndAppId returns a protected copy', async () => {
  const storage = createInMemoryStorage();
  const todoRepo = createInMemoryTodoRepository(storage);
  
  await todoRepo.save(makeTodo('t1', 'a1', 'Original Title'));
  const found = await todoRepo.findActiveByIdAndAppId('t1', 'a1');
  
  if (found) {
    found.completed = true; // caller が改変
    found.title = 'Hacked';
  }
  
  // 再取得で確認
  const found2 = await todoRepo.findActiveByIdAndAppId('t1', 'a1');
  expect(found2?.completed).toBe(false);
  expect(found2?.title).toBe('Original Title');
});
```

---

## 防御的コピーテストの成果

防御的コピーテストにより：

1. **ストア整合性が保証される** — caller の改変がストアに影響しない
2. **実装習慣の統一** — in-memory 実装から本物の実装への移行が安全
3. **境界の明確化** — repository は「ストアのゲートウェイ」として機能
4. **バグ検出** — 防御し忘れたフィールドが即座に検出される

---

## まとめ

防御的コピーテストは、「当たり前に正しい」と思われた in-memory repository の実装が、実は**参照漏れによる汚染の危険性**を持っていることを実証する。

特に **cascade delete** のような複雑な操作で、複数リポジトリが同じ storage を参照している場合、防御的コピーなしでは予測不可能なバグが発生する。

このプロジェクトで防御的コピーテストを徹底することで、将来 MySQL 実装に移行する際の「参照実装」として機能し、チーム全体の実装品質向上につながる。
