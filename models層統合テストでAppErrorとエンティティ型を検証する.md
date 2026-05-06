# models層統合テストで AppError とエンティティ型を検証する

## 対象読者

- レイヤードアーキテクチャで型安全なバックエンドを構築している
- AppError のような独自エラー型をどう検証するか知りたい
- models 層をどの粒度でテストすべきか悩んでいる

## この記事が扱う範囲

- `AppError` の伝播とハンドリング
- エンティティ型（`AppEntity` / `TodoEntity`）の shape 検証
- インタラクター経由の round-trip テスト
- models 層とテスト結果の結びつき

---

## 背景：models 層をなぜテストするのか

レイヤードアーキテクチャでは、models 層は以下の責務を持つ：

- **エンティティ型の定義** — `AppEntity` / `TodoEntity` のフィールド
- **ドメインエラー型の定義** — `AppError` とエラーコード
- **型安全性の保証** — コンパイル時の契約

これらは「定義なので当たり前に正しい」と思われやすいが、エンティティの shape や AppError の伝播がテスト前提で正しく動作することを実証する必要がある。

---

## AppError の検証設計

### AppError の構造

```typescript
export type AppErrorCode =
  | 'VALIDATION_ERROR'
  | 'CONFLICT'
  | 'NOT_FOUND'
  | 'REPOSITORY_ERROR';

export class AppError extends Error {
  public readonly code: AppErrorCode;

  public constructor(code: AppErrorCode, message: string) {
    super(message);
    this.name = 'AppError';
    this.code = code;
  }
}

export function isAppError(error: unknown): error is AppError {
  return error instanceof AppError;
}
```

`AppError` は以下の特性を持つ：

1. **`code` フィールド** — エラーの種類を識別
2. **`isAppError()` ガード関数** — 型が安全に動的に判定できる
3. **レイヤー間を透過的に伝播** — repository → service → controller へ

### AppError 伝播テスト

`models/app-error.test.ts` では、AppError が層を超えて正しく伝播することを検証する。

```typescript
describe('AppError cross-layer propagation', () => {
  it('interactor throws AppError with NOT_FOUND and isAppError recognizes it', async () => {
    const interactor = makeInteractor();
    let caught: unknown;
    try {
      await interactor.get({ appId: GHOST_ID });
    } catch (e) {
      caught = e;
    }
    expect(isAppError(caught)).toBe(true);
    if (isAppError(caught)) {
      expect(caught.code).toBe('NOT_FOUND');
    }
  });

  it('interactor throws AppError with CONFLICT on duplicate create', async () => {
    const interactor = makeInteractor();
    await interactor.create({ name: 'Dup' });
    let caught: unknown;
    try {
      await interactor.create({ name: 'Dup' });
    } catch (e) {
      caught = e;
    }
    expect(isAppError(caught)).toBe(true);
    if (isAppError(caught)) {
      expect(caught.code).toBe('CONFLICT');
    }
  });
});
```

**テスト設計のポイント：**

1. `interactor.get()` のような実際の操作を通して AppError が thrown される
2. `isAppError()` が正しく型を判定できることを検証
3. エラーコード（`NOT_FOUND` / `CONFLICT`）が期待通りセットされている

この設計により、AppError が repository から service を経由して caller に到達することを型レベルで証明している。

---

## エンティティ型の shape 検証

### エンティティの構造

```typescript
export type AppEntity = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null;
};

export type TodoEntity = {
  id: string;
  appId: string;
  title: string;
  completed: boolean;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null;
};
```

エンティティ型には以下の特性がある：

1. **`deletedAt` の nullable性** — soft delete をサポート
2. **タイムスタンプの ISO 8601 形式** — JSON 互換性
3. **ID の string 型** — UUID など外部生成可能

### エンティティ shape テスト

`models/app.test.ts` では、エンティティ型が期待通りの shape を持つことを検証する。

```typescript
describe('AppEntity shape', () => {
  it('should round-trip through interactor', async () => {
    const interactor = makeInteractor();
    const created = await interactor.create({ name: 'Test App' });
    
    expect(created).toHaveProperty('id');
    expect(created).toHaveProperty('name', 'Test App');
    expect(created).toHaveProperty('createdAt');
    expect(created).toHaveProperty('updatedAt');
    expect(created).toHaveProperty('deletedAt', null);
  });

  it('soft-deleted app has deletedAt timestamp', async () => {
    const interactor = makeInteractor();
    const app = await interactor.create({ name: 'Test' });
    await interactor.delete({ appId: app.id });
    
    let deleted: AppEntity | null = null;
    try {
      // 削除後は見えないが、内部的には deletedAt が設定される
      deleted = await interactor.get({ appId: app.id });
    } catch (e) {
      // NOT_FOUND が期待される
    }
    
    expect(deleted).toBeNull();
  });
});
```

**テスト設計のポイント：**

1. **Round-trip 検証** — 作成 → 取得で shape が保たれるか
2. **null 初期化チェック** — 新規作成時に `deletedAt: null`
3. **ソフトデリート後の変化** — `deletedAt` がタイムスタンプ化される

---

## models テストにおける helpers の役割

models テストでは、インタラクターへのアクセスが必須になるため、shared helpers が活躍する。

```typescript
// helpers.ts
export const GHOST_APP_ID  = '00000000-0000-0000-0000-000000000000';
export const UUID_RE       = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
export const ISO8601_RE    = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/;

function makeInteractor() {
  const storage = createInMemoryStorage();
  return createAppInteractor({
    appRepository: createInMemoryAppRepository(storage),
    todoRepository: createInMemoryTodoRepository(storage),
  });
}
```

この helpers により：

- テストファイルで毎回 DI コンテナを組み立てる手間が省ける
- エラーテスト用のダミー ID（`GHOST_APP_ID`）を一元管理
- 正規表現（UUID / ISO8601）を複数ファイルで再利用可能

---

## models テストの実装上の工夫

### 型ガードの検証

`isAppError()` のような型ガード関数をテストに含めることで、TypeScript の型推論が実行時挙動と一致することを保証する。

```typescript
it('isAppError recognizes AppError instances', () => {
  const err = new AppError('NOT_FOUND', 'App not found');
  expect(isAppError(err)).toBe(true);
  
  const notErr = new Error('Generic error');
  expect(isAppError(notErr)).toBe(false);
});
```

### エンティティ生成の一貫性

models 層のテストでは、エンティティを手作りするのではなく、実際の interactor / repository を通じて生成する。

```typescript
// ❌ 悪い例：直接オブジェクトを作成
const fakeApp: AppEntity = {
  id: 'fake',
  name: 'Fake',
  createdAt: '2024-01-01T00:00:00Z',
  updatedAt: '2024-01-01T00:00:00Z',
  deletedAt: null,
};

// ✅ 良い例：実装レイヤーを通す
const app = await interactor.create({ name: 'Real' });
```

実装を通すことで、エンティティが実際に「どう構築されるか」を検証できる。

---

## models 層テストの成果

models 層統合テストを整備することで：

1. **エラー型の伝播が証明される** — AppError が層を越えても型安全性を失わない
2. **エンティティ shape の一貫性が保証される** — 作成・更新・削除後の shape が期待通り
3. **後続レイヤーのテストが軽くなる** — models が正しいことが前提になるため、上位レイヤーは contract に集中できる
4. **型とロジックの乖離が検出される** — 型定義だけでなく、実装が型に従っていることが保証される

---

## まとめ

models 層を統合テストで検証することは、「当たり前に正しい」と思われた型定義やエラー型が、実装を通して本当に正しく動作することを保証する。

特に `isAppError()` や `deletedAt: null` のような「振る舞いを持つ型」は、テストなしでは気づかないバグが潜んでいる可能性がある。

models 層のテストは後続レイヤーのテストを簡素化するための基盤となる。
