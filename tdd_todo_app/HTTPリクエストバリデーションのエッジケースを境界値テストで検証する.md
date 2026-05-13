# HTTP request validation のエッジケース：100/200字境界と boolean型チェック

## 対象読者

- Express / Hono でリクエスト validation を実装している
- 境界値テスト（boundary value test）の重要性を理解したい
- boolean 型の誤検証（"false" 文字列が true に変換されてしまう問題）を避けたい

## この記事が扱う範囲

- `validateName()` / `validateTitle()` のロジック
- 100 文字 / 200 文字の境界値テスト
- 空文字 vs 空白のみの区別
- boolean 型の厳密なチェック
- `toRecord()` による malformed body の処理

---

## 背景：バリデーション層の重要性

HTTP request validation は、アプリケーションの**最初の防衛線**である。

- **type safety** — 文字列 / 数値 / boolean の型を厳密に
- **business rule** — 文字数制限、一意制約
- **security** — 不正な形式の検却、injection 対策

テストなしでは、以下のような subtle バグが潜んでいる：

- `"false"` 文字列が `true` に変換される
- 100 文字ちょうどが reject されたり accept されたりする
- 空白のみの name が accept される

---

## validateName / validateTitle の設計

```typescript
function validateName(value: unknown): string {
  if (typeof value !== 'string') {
    throw new AppError('VALIDATION_ERROR', 'name is required');
  }

  const trimmed = value.trim();
  if (trimmed.length === 0) {
    throw new AppError('VALIDATION_ERROR', 'name must not be empty');
  }

  if (trimmed.length > 100) {
    throw new AppError(
      'VALIDATION_ERROR',
      'name must be at most 100 characters',
    );
  }

  return trimmed; // ← 重要：trim() 後の値を返す
}

function validateTitle(value: unknown): string {
  // validateName と同様だが、制限は 200 文字
  const trimmed = value.trim();
  if (trimmed.length > 200) {
    throw new AppError('VALIDATION_ERROR', 'title must be at most 200 characters');
  }
  return trimmed;
}
```

**設計のポイント：**

1. **型チェック** — 非 string は即座に reject
2. **trim + 空白チェック** — `"   "` は `""` になり reject
3. **文字数制限** — `length > max` で上限を定義
4. **trim 後の値を返す** — ストレージには trim 済み値を保存

---

## 境界値テスト（boundary value testing）

### 100 文字の境界値

```typescript
describe('parseCreateAppInput', () => {
  it('throws VALIDATION_ERROR when name exceeds 100 characters', () => {
    expect(() => parseCreateAppInput({ name: 'a'.repeat(101) })).toThrow(
      expect.objectContaining({ code: 'VALIDATION_ERROR' }),
    );
  });

  it('accepts a name of exactly 100 characters', () => {
    const result = parseCreateAppInput({ name: 'a'.repeat(100) });
    expect(result.name).toHaveLength(100);
  });
});
```

**境界値テストの原理：**

| テストケース | 期待値 | 理由 |
|---|---|---|
| 99 文字 | ✅ accept | 上限以下 |
| **100 文字** | ✅ accept | 上限ちょうど（重要） |
| **101 文字** | ❌ reject | 上限超過（重要） |

境界値テスト（on-boundary / off-boundary）は、条件判定のバグを最も効率よく検出できる。

### 200 文字の境界値

```typescript
describe('parseCreateTodoInput', () => {
  it('accepts title of exactly 200 characters', () => {
    const result = parseCreateTodoInput(APP_ID, { title: 'a'.repeat(200) });
    expect(result.title).toHaveLength(200);
  });

  it('throws VALIDATION_ERROR when title exceeds 200 characters', () => {
    expect(() => parseCreateTodoInput(APP_ID, { title: 'a'.repeat(201) })).toThrow(
      expect.objectContaining({ code: 'VALIDATION_ERROR' }),
    );
  });
});
```

---

## 空文字 vs 空白のみの区別

### trim() による正規化

```typescript
it('throws VALIDATION_ERROR when name is whitespace only', () => {
  expect(() => parseCreateAppInput({ name: '   ' })).toThrow(
    expect.objectContaining({ code: 'VALIDATION_ERROR' }),
  );
});

it('returns trimmed name with surrounding whitespace', () => {
  expect(parseCreateAppInput({ name: '  My App  ' })).toEqual({ name: 'My App' });
});
```

**テスト設計のポイント：**

1. **trim() 前の値** — `"   "` のような空白のみ
2. **trim() 後の判定** — `"   ".trim() === ""` で reject
3. **trim() 後の返却** — ストレージには `"My App"` を保存

このメカニズムにより、以下が保証される：

- `"  My App  "` と `"My App"` が同じものとして扱われる
- 一意制約チェック（`existsActiveByName`）で重複判定が正確

---

## Boolean 型の厳密なチェック

### Boolean 誤検証の歴史的背景

```typescript
// ❌ 悪い例：Boolean() コンストラクタを使った型変換
const completed = Boolean(body.completed);

// 結果：
// Boolean("false") === true (文字列は truthy)
// Boolean(0) === false
// Boolean("") === false
// Boolean(null) === false
```

この問題は、review フェーズで実際に検出されたバグである（diary 参照）。

### 厳密な boolean チェック

```typescript
export function parseUpdateTodoInput(
  appId: string,
  todoId: string,
  body: unknown,
): UpdateTodoInput {
  const payload = toRecord(body);
  const input: UpdateTodoInput = { appId, todoId };

  if (payload.completed !== undefined) {
    if (typeof payload.completed !== 'boolean') {
      throw new AppError('VALIDATION_ERROR', 'completed must be a boolean');
    }
    input.completed = payload.completed; // 型チェック後に代入
  }

  return input;
}
```

**ポイント：**

1. **typeof チェック** — `typeof payload.completed === 'boolean'`
2. **型チェック失敗 → reject** — string / number / null はすべて reject
3. **チェック後に代入** — 型安全性の確保

### Boolean 検証テスト

```typescript
describe('parseUpdateTodoInput', () => {
  it('accepts completed as true', () => {
    const result = parseUpdateTodoInput(APP_ID, TODO_ID, { completed: true });
    expect(result.completed).toBe(true);
  });

  it('accepts completed as false', () => {
    const result = parseUpdateTodoInput(APP_ID, TODO_ID, { completed: false });
    expect(result.completed).toBe(false);
  });

  it('throws VALIDATION_ERROR when completed is the string "false"', () => {
    expect(() => parseUpdateTodoInput(APP_ID, TODO_ID, { completed: 'false' })).toThrow(
      expect.objectContaining({ code: 'VALIDATION_ERROR' }),
    );
  });

  it('throws VALIDATION_ERROR when completed is 0', () => {
    expect(() => parseUpdateTodoInput(APP_ID, TODO_ID, { completed: 0 })).toThrow(
      expect.objectContaining({ code: 'VALIDATION_ERROR' }),
    );
  });

  it('throws VALIDATION_ERROR when completed is 1', () => {
    expect(() => parseUpdateTodoInput(APP_ID, TODO_ID, { completed: 1 })).toThrow(
      expect.objectContaining({ code: 'VALIDATION_ERROR' }),
    );
  });

  it('throws VALIDATION_ERROR when completed is null', () => {
    expect(() => parseUpdateTodoInput(APP_ID, TODO_ID, { completed: null })).toThrow(
      expect.objectContaining({ code: 'VALIDATION_ERROR' }),
    );
  });
});
```

---

## toRecord の設計：malformed body 対応

```typescript
function toRecord(value: unknown): Record<string, unknown> {
  if (typeof value !== 'object' || value === null || Array.isArray(value)) {
    return {}; // 空のオブジェクトを返す
  }

  return value as Record<string, unknown>;
}
```

**目的：** JSON パース後の値が、想定外の形式（配列・null・プリミティブ）でも graceful に処理

```typescript
it('returns empty name from null body', () => {
  expect(() => parseCreateAppInput(null)).toThrow(
    expect.objectContaining({ code: 'VALIDATION_ERROR' }),
  );
});

it('throws VALIDATION_ERROR when body is an array', () => {
  expect(() => parseCreateAppInput([{ name: 'x' }])).toThrow(
    expect.objectContaining({ code: 'VALIDATION_ERROR' }),
  );
});

it('throws VALIDATION_ERROR when body is a string', () => {
  expect(() => parseCreateAppInput('name')).toThrow(
    expect.objectContaining({ code: 'VALIDATION_ERROR' }),
  );
});
```

---

## Trim の一貫性：バリデーション時と保存時

### レビューで見つかった問題

当初の実装では、バリデーション時にはトリムするが、保存時には未トリムの値を保存していた。

```typescript
// ❌ 問題のあった実装
if (duplicated) throw new AppError('CONFLICT', 'name already exists');
const name_original = body.name; // trim していない
// ...
const app: AppEntity = { name: name_original, ... };
await appRepository.save(app);
```

結果：

- バリデーション → `"  My App  "` を許可（trim 後 "My App"）
- 保存 → `"  My App  "` をそのまま保存
- 次回検索 → `existsActiveByName("My App")` が重複を見落とす

### 修正版

```typescript
// ✅ 修正後：保存前に trim()
const name = (body.name as string).trim();
const app: AppEntity = { name, ... };
```

これにより、バリデーション層と repository 層の動作が一貫する。

---

## HTTP request validation テストの成果

HTTP validation テストにより：

1. **境界値がテストされる** — off-by-one エラーの検出
2. **型の厳密性が保証される** — Boolean("false") のような誤変換を防止
3. **malformed body の graceful 処理** — null / array 等への対応
4. **ビジネスロジック層への信頼** — validation を通った値は安全

---

## まとめ

HTTP request validation は、ビジネスロジック層が「正しい形式のデータだけを扱う」ことを保証する責務を持つ。

特に重要な工夫：

1. **型チェック** — `typeof` による厳密な型判定
2. **境界値テスト** — 100 / 200 字ちょうどの検証
3. **trim の一貫性** — バリデーション・保存・検索で同じ正規化を適用
4. **malformed body への対応** — 想定外の形式を graceful に reject

これらの工夫により、後続のビジネスロジック層が「validate 済みの安全なデータ」だけを扱えるようになり、結果として system 全体の品質が向上する。
