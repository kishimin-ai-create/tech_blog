# in-memory リポジトリの防御的コピーテスト

## 対象読者

- TypeScript でテスト用のインメモリリポジトリを実装している
- テスト間でデータが汚染される問題を経験したことがある
- `expect(found).toEqual(app)` と `expect(found).not.toBe(app)` の違いが気になっている

---

## 問題：参照を返すと何が起きるか

インメモリリポジトリの素朴な実装は `Map<string, Entity>` に保存したオブジェクトをそのまま返す。

```typescript
// 問題のある実装（参照を返す）
async function findActiveById(id: string): Promise<AppEntity | null> {
  return storage.apps.get(id) ?? null;  // ← Map の中身を直接返す
}
```

このとき、呼び出し側がオブジェクトを変更すると `Map` の中身も変わる。

```typescript
const found = await repo.findActiveById('app-1');
found.name = 'Mutated';  // ← これがストアに直接影響する

const found2 = await repo.findActiveById('app-1');
console.log(found2.name);  // 'Mutated' ← ストアが汚染されている
```

テスト環境では `beforeEach` でリポジトリを再生成するが、同一テストケース内で複数回 `findActiveById` を呼ぶシナリオでは汚染が起きる。本番環境でも、ユースケース層がエンティティを変更してから保存し忘れたとき、ストアが不整合な状態になるリスクがある。

---

## 解決策：防御的コピー（Defensive Copy）

このプロジェクトの実装は、読み書き両方でスプレッドコピーを行う。

```typescript
// src/infrastructure/in-memory-repositories.ts

function cloneApp(app: AppEntity): AppEntity {
  return { ...app };
}

async function save(app: AppEntity): Promise<void> {
  storage.apps.set(app.id, cloneApp(app));  // ← 保存時にコピー
}

async function findActiveById(id: string): Promise<AppEntity | null> {
  const app = storage.apps.get(id);
  return app && app.deletedAt === null ? cloneApp(app) : null;  // ← 返却時にコピー
}
```

**保存時のコピー**：呼び出し側が渡したオブジェクトをそのまま格納せず、コピーを格納する。呼び出し側がその後にオブジェクトを変更してもストアは影響を受けない。

**返却時のコピー**：ストア内のオブジェクトをそのまま返さず、コピーを返す。呼び出し側がオブジェクトを変更してもストアは影響を受けない。

---

## テストで検証する

### ユニットテスト（`src/infrastructure/in-memory-repositories.test.ts`）

```typescript
it('returns a copy, not the original reference', async () => {
  const app = makeApp();
  await repo.save(app);
  const found = await repo.findActiveById(app.id);
  expect(found).not.toBe(app);  // ← 同一参照ではない
});
```

`not.toBe()` は参照の同一性を確認する。`toEqual()` は「同じ値を持つ」ことを確認するが、参照が同じかどうかは問わない。ここで使うべきは `not.toBe()` だ。

`toEqual` と `not.toBe` を両方書くと、「値は同じだが、参照は別」という意図が明確になる。

```typescript
it('returns a copy with the same values', async () => {
  const app = makeApp();
  await repo.save(app);
  const found = await repo.findActiveById(app.id);
  expect(found).toEqual(app);   // 値は同じ
  expect(found).not.toBe(app);  // 参照は別
});
```

### 統合テスト（`src/tests/integrations/infrastructure/in-memory-repositories.test.ts`）

統合テストでは「返却されたオブジェクトを変更してもストアが汚染されないこと」を確認する。

```typescript
describe('repository returns clones (no mutation through returned reference)', () => {
  it('mutating a returned app entity does not affect stored data', async () => {
    const storage = createInMemoryStorage();
    const appRepo = createInMemoryAppRepository(storage);
    await appRepo.save(makeApp('a1', 'Original'));

    const found = await appRepo.findActiveById('a1');
    if (found) found.name = 'Mutated';  // ← 返却値を変更

    const found2 = await appRepo.findActiveById('a1');
    expect(found2?.name).toBe('Original');  // ← ストアは変わっていない
  });

  it('mutating a returned todo entity does not affect stored data', async () => {
    const storage = createInMemoryStorage();
    const todoRepo = createInMemoryTodoRepository(storage);
    await todoRepo.save(makeTodo('t1', 'a1', 'Original Title'));

    const found = await todoRepo.findActiveById('a1', 't1');
    if (found) found.title = 'Mutated';  // ← 返却値を変更

    const found2 = await todoRepo.findActiveById('a1', 't1');
    expect(found2?.title).toBe('Original Title');  // ← ストアは変わっていない
  });
});
```

このテストが通らない実装（参照を直接返す実装）では、`found2?.name` は `'Mutated'` になる。

---

## `listActive()` でも防御的コピーが必要

単一エンティティの取得だけでなく、リスト返却でも同様だ。

```typescript
async function listActive(): Promise<AppEntity[]> {
  return withRepositoryError(() =>
    [...storage.apps.values()]
      .filter(app => app.deletedAt === null)
      .map(cloneApp),  // ← 各要素をコピー
  );
}
```

`[...storage.apps.values()]` でイテレータを配列に展開するが、要素のコピーはしていない。`.map(cloneApp)` がないと、リスト内の各要素はストアへの参照になる。

---

## なぜスプレッドコピーで十分か

このプロジェクトのエンティティはすべてネストのないフラットなオブジェクトだ。

```typescript
export type AppEntity = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null;
};
```

すべてのプロパティがプリミティブ型（`string` / `null`）なので、`{ ...app }` の浅いコピーで完全な複製になる。ネストしたオブジェクトや配列を含む場合は深いコピーが必要になる点に注意すること。

---

## まとめ

- リポジトリは保存時・返却時の両方でエンティティをコピーする（防御的コピー）
- `not.toBe()` で「参照の独立性」を確認し、`toEqual()` で「値の同一性」を確認する
- 統合テストでは「返却値を変更してもストアが汚染されないこと」を実際の操作で検証する
- エンティティがフラットな構造であれば `{ ...entity }` の浅いコピーで十分だが、ネストがある場合は深いコピーが必要
