# completed フィールドの boolean 厳密バリデーション — typeof で 'yes' や 1 を弾く

## エラーの概要

Todo の `completed` フィールド更新時、`completed: 'yes'` や `completed: 1` のような「truthy な非 boolean」値を受け付けてしまうバグがあった。ユースケース層に渡ってくる値が `boolean` である保証がなく、型定義と実際の動作が乖離していた。

---

## 原因

バリデーションを追加する前の実装では、`completed` フィールドの存在確認しかしていなかった。

```typescript
// ❌ 問題のある実装（型チェックなし）
if (payload.completed !== undefined) {
  input.completed = payload.completed as boolean;  // ← 型アサーションで誤魔化している
}
```

JSON リクエストボディには `'yes'`, `1`, `"true"`, `null` など様々な値が入りうる。TypeScript の型アサーション（`as boolean`）は実行時には何もしない。コンパイルを通過させるための注釈にすぎず、実際の値は `string` や `number` のままリポジトリに届いていた。

---

## 修正

```typescript
// ✅ 修正後の実装（typeof で厳密に検証）
if (payload.completed !== undefined) {
  if (typeof payload.completed !== 'boolean') {
    throw new AppError('VALIDATION_ERROR', 'completed must be a boolean');
  }
  input.completed = payload.completed;
}
```

`typeof payload.completed !== 'boolean'` の条件で `true` / `false` 以外をすべて拒否する。型アサーションを使わないため、TypeScript の型推論が `input.completed` を `boolean` と正しく認識する。

---

## テストで検証するエッジケース

### 正常系

```typescript
it('returns UpdateTodoInput with completed field', () => {
  const result = parseUpdateTodoInput('app-1', 'todo-1', { completed: true });
  expect(result).toEqual({ appId: 'app-1', todoId: 'todo-1', completed: true });
});

it('accepts completed: false', () => {
  const result = parseUpdateTodoInput('app-1', 'todo-1', { completed: false });
  expect(result.completed).toBe(false);
});
```

`false` を明示的にテストすることが重要だ。バリデーションが「truthy チェック」になっていると、`completed: false` が `undefined` と同様に扱われてフィールドが消えてしまう。

### 異常系：文字列

```typescript
it('throws VALIDATION_ERROR when completed is not a boolean', () => {
  expect(() =>
    parseUpdateTodoInput('app-1', 'todo-1', { completed: 'yes' }),
  ).toThrow(expect.objectContaining({ code: 'VALIDATION_ERROR' }));
});
```

`'yes'` は JavaScript の真偽値評価では truthy だが、`typeof 'yes' === 'string'` なので弾かれる。`'true'` も同様だ。

### 異常系：数値

```typescript
it('throws VALIDATION_ERROR when completed is a number', () => {
  expect(() =>
    parseUpdateTodoInput('app-1', 'todo-1', { completed: 1 }),
  ).toThrow(expect.objectContaining({ code: 'VALIDATION_ERROR' }));
});
```

`1` は C 言語的な慣習では「true」だが、TypeScript の `boolean` ではない。JSON API の仕様として `true` / `false` のみを受け付けるなら、`1` も弾くべきだ。

---

## なぜ `Boolean(value)` ではダメか

```typescript
// ❌ これは不十分
input.completed = Boolean(payload.completed);
```

`Boolean('yes')` は `true` を返す。`Boolean(0)` は `false` を返す。これでは「文字列や数値でも完了フラグを操作できる」ことになり、API の仕様として曖昧になる。

`typeof !== 'boolean'` を使うことで、JSON の `true` と `false` だけを受け付けるという仕様が明確になる。

---

## `completed` を省略した場合

```typescript
it('returns base input when body has no fields', () => {
  const result = parseUpdateTodoInput('app-1', 'todo-1', {});
  expect(result).toEqual({ appId: 'app-1', todoId: 'todo-1' });
});
```

`completed` が含まれていない場合は、フィールド自体を `input` に設定しない。これにより、ユースケース層で「`undefined` の場合は既存の値を維持する」という挙動が正しく動く。

もし `completed: undefined` を代入してしまうと、ユースケース層で `input.completed !== undefined` の判定が意図通りに動かなくなる可能性がある。

---

## バリデーション実装全体

```typescript
export function parseUpdateTodoInput(
  appId: string,
  todoId: string,
  body: unknown,
): UpdateTodoInput {
  const payload = toRecord(body);
  const input: UpdateTodoInput = { appId, todoId };

  if (payload.title !== undefined) {
    input.title = validateTitle(payload.title);
  }

  if (payload.completed !== undefined) {
    if (typeof payload.completed !== 'boolean') {
      throw new AppError('VALIDATION_ERROR', 'completed must be a boolean');
    }
    input.completed = payload.completed;
  }

  return input;
}
```

`title` と `completed` はどちらも任意フィールドだが、バリデーションの方法が異なる。`title` は文字列の検証（長さ・空白）が必要なのでヘルパー関数 `validateTitle()` に委ねる。`completed` はプリミティブ型の確認だけなので `typeof` チェックで十分だ。

---

## まとめ

- `typeof payload.completed !== 'boolean'` で `'yes'`、`1`、`'true'`、`null` などを明確に弾く
- `false` の受け入れを明示的にテストする（truthy チェックではない証拠）
- `Boolean(value)` への変換は「型を変換する」行為であり、API の入力バリデーションとしては不適切
- `completed` が省略された場合はフィールドを設定せず、ユースケース層での「省略 = 変更なし」処理に委ねる
