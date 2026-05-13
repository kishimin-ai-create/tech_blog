# repositories層統合テストでリポジトリ契約を検証する

## 対象読者

- Clean Architecture / Hexagonal Architecture でリポジトリインターフェースを定義している
- リポジトリの契約（contract）をテストする方法を知りたい
- in-memory 実装をテストして、本物の DB 実装の基準を作りたい

## この記事が扱う範囲

- `AppRepository` / `TodoRepository` インターフェースの契約定義
- soft-delete フィルタリングの検証
- 防御的コピーの検証（返却値の不変性）
- 複合条件テスト（unique 制約・soft-delete の組み合わせ）

---

## 背景：リポジトリテストなしに実装は進められない

リポジトリ層は以下の責務を持つ：

- **データの永続化** — `save()` で entity を保存
- **ソフトデリート対応** — `deletedAt` が null でないレコードは「見えない」状態
- **一意制約** — `existsActiveByName()` で name 一意性をチェック
- **防御的コピー** — 返却データの改変が内部ストアに影響しない

これらの責務は「単にインターフェースを定義した」だけでは実装品質が担保されない。**統合テストで契約を実証する**ことが必須。

---

## AppRepository インターフェースの設計

```typescript
export interface AppRepository {
  save(app: AppEntity): Promise<void>;
  listActive(): Promise<AppEntity[]>;
  findActiveById(id: string): Promise<AppEntity | null>;
  existsActiveByName(name: string, excludeId?: string): Promise<boolean>;
}
```

各メソッドの責務：

| メソッド | 責務 | テスト項目 |
|---|---|---|
| `save(app)` | 作成 or 更新 | 初回 save・上書き・id 一致時の更新 |
| `listActive()` | 活性化したすべてを取得 | 空配列・ソフトデリート後の非表示 |
| `findActiveById(id)` | ID で単体取得（活性化のみ） | 存在・非存在・ソフトデリート後 null |
| `existsActiveByName(name, excludeId?)` | name の一意性チェック | 存在・非存在・削除状態・自己除外 |

---

## AppRepository 契約テストの実装

### save + findActiveById の検証

```typescript
describe('AppRepository contract', () => {
  let repo: AppRepository;

  beforeEach(() => {
    repo = createInMemoryAppRepository(createInMemoryStorage());
  });

  describe('save + findActiveById', () => {
    it('persisted entity is retrievable by id', async () => {
      await repo.save(makeApp('app-1', 'My App'));
      const found = await repo.findActiveById('app-1');
      expect(found?.id).toBe('app-1');
      expect(found?.name).toBe('My App');
    });

    it('returns null for an unknown id', async () => {
      expect(await repo.findActiveById('unknown')).toBeNull();
    });

    it('returns null for a soft-deleted entity', async () => {
      await repo.save(makeApp('app-1', 'Deleted', TIME));
      expect(await repo.findActiveById('app-1')).toBeNull();
    });

    it('overwriting with the same id updates the entity', async () => {
      await repo.save(makeApp('app-1', 'Original'));
      await repo.save(makeApp('app-1', 'Updated'));
      expect((await repo.findActiveById('app-1'))?.name).toBe('Updated');
    });

    it('returned entity is a defensive copy, not the original reference', async () => {
      const app = makeApp('app-1', 'Copy Test');
      await repo.save(app);
      const found = await repo.findActiveById('app-1');
      expect(found).not.toBe(app);
    });
  });
});
```

**テスト設計のポイント：**

1. **基本的な save / find** — 最小限のハッピーパス
2. **null 返却** — 存在しないデータと削除済みデータの両方をテスト
3. **上書き動作** — 同じ ID で save を2回呼ぶと更新される
4. **防御的コピー** — `found !== app` で参照が異なることを検証

防御的コピーの検証は特に重要。もし repository が内部参照をそのまま返していたら、caller が返却物を改変すると store が汚染される。

### listActive の検証

```typescript
describe('listActive', () => {
  it('returns empty array when no entities exist', async () => {
    expect(await repo.listActive()).toEqual([]);
  });

  it('returns only active (non-deleted) entities', async () => {
    await repo.save(makeApp('app-1', 'Active'));
    await repo.save(makeApp('app-2', 'Deleted', TIME));
    const list = await repo.listActive();
    expect(list).toHaveLength(1);
    expect(list[0].name).toBe('Active');
  });

  it('returns all active entities', async () => {
    await repo.save(makeApp('app-1', 'A'));
    await repo.save(makeApp('app-2', 'B'));
    expect(await repo.listActive()).toHaveLength(2);
  });
});
```

**テスト設計のポイント：**

1. **空配列を返すケース** — データなしで正常に動作する
2. **ソフトデリート後のフィルタリング** — `deletedAt !== null` は一覧に含まれない
3. **複数データの取得** — N 個のデータを正しく返せる

### existsActiveByName の検証

```typescript
describe('existsActiveByName', () => {
  it('returns true when an active entity with the given name exists', async () => {
    await repo.save(makeApp('app-1', 'My App'));
    expect(await repo.existsActiveByName('My App')).toBe(true);
  });

  it('returns false when no entity with the name exists', async () => {
    expect(await repo.existsActiveByName('None')).toBe(false);
  });

  it('returns false when only a soft-deleted entity has the name', async () => {
    await repo.save(makeApp('app-1', 'Gone', TIME));
    expect(await repo.existsActiveByName('Gone')).toBe(false);
  });

  it('returns false when the only matching entity is excluded by id (self-check)', async () => {
    await repo.save(makeApp('app-1', 'Same Name'));
    expect(await repo.existsActiveByName('Same Name', 'app-1')).toBe(false);
  });

  it('returns true when another entity (not excluded by id) has the same name', async () => {
    await repo.save(makeApp('app-1', 'Shared'));
    await repo.save(makeApp('app-2', 'Shared'));
    expect(await repo.existsActiveByName('Shared', 'app-1')).toBe(true);
  });
});
```

**テスト設計のポイント：**

1. **基本的な存在チェック** — name が存在する → true
2. **非存在と削除済みの区別** — ソフトデリートされたレコードは存在しない扱い
3. **自己除外フィルタ** — 更新処理では `excludeId` で自分以外の重複を確認
4. **複合条件** — excludeId と soft-delete の両方が有効

---

## TodoRepository の多層テスト

TodoRepository には AppRepository と異なる特性がある：

```typescript
export interface TodoRepository {
  save(todo: TodoEntity): Promise<void>;
  listActiveByAppId(appId: string): Promise<TodoEntity[]>;
  findActiveByIdAndAppId(id: string, appId: string): Promise<TodoEntity | null>;
  existsActiveByAppIdAndTitle(appId: string, title: string, excludeId?: string): Promise<boolean>;
}
```

app scoping が加わる：

```typescript
describe('TodoRepository appId-scoped contract', () => {
  it('listActiveByAppId returns only todos for the specified app', async () => {
    const storage = createInMemoryStorage();
    const todoRepo = createInMemoryTodoRepository(storage);

    await todoRepo.save(makeTodo('t1', 'app-1', 'Todo 1'));
    await todoRepo.save(makeTodo('t2', 'app-2', 'Todo 2'));

    const app1Todos = await todoRepo.listActiveByAppId('app-1');
    expect(app1Todos).toHaveLength(1);
    expect(app1Todos[0].title).toBe('Todo 1');
  });

  it('findActiveByIdAndAppId checks both id and appId', async () => {
    const storage = createInMemoryStorage();
    const todoRepo = createInMemoryTodoRepository(storage);

    await todoRepo.save(makeTodo('t1', 'app-1', 'Todo'));

    // 存在する ID だが app-2 には属さない
    const result = await todoRepo.findActiveByIdAndAppId('t1', 'app-2');
    expect(result).toBeNull();
  });
});
```

---

## リポジトリ契約テストの成果

リポジトリ層を統合テストで検証することで：

1. **メソッドの責務が明文化される** — インターフェース定義だけでなく、期待動作を実装側に強制
2. **ソフトデリート・スコーピング・一意性などの複合条件が証明される** — 複数の responsibility が同時に満たされることを検証
3. **防御的コピーが実装される** — 返却データの安全性が保証される
4. **本物の DB 実装の基準になる** — 将来 MySQL / PostgreSQL 実装に移行する際、同じテストで検証可能

---

## まとめ

リポジトリ層統合テストは、インターフェース定義だけでは見逃される細部（ソフトデリート・スコーピング・防御的コピー）を網羅的にテストする。

特に **`existsActiveByName` の `excludeId` パラメータ** や **複数条件の組み合わせ** は、テストなしでは実装漏れが検出されない。

リポジトリテストは、架空のデータベース実装から本物の実装への移行をスムーズにするための「参照実装」となる。
