# Zod スキーマをバリデーションと DTO 型の単一情報源にリファクタリングした

## 対象読者

- TypeScript + Hono でバックエンド API を実装しているエンジニア
- Zod スキーマを定義済みだが、バリデーションロジックと型定義が別々に管理されている現状に課題を感じている人

---

## 問題の背景

`tddTodoApp` バックエンドのコントローラー層では、Zod スキーマ（`schemas.ts`）が既に定義されていた。しかしリファクタリング前の `request-validation.ts` は、そのスキーマを使わず **独自のバリデーションヘルパー** を手書きで維持していた。

```
backend/src/controllers/
  schemas.ts              ← Zod スキーマ定義（ルールの定義元）
  request-validation.ts   ← 手書きバリデーション（ルールの再実装）  ← 重複
  http-presenter.ts       ← 手書き AppDto/TodoDto インターフェース  ← 重複
```

具体的には次の 3 つのヘルパー関数が `request-validation.ts` に存在していた。

| ヘルパー | 役割 |
|---|---|
| `toRecord()` | `unknown` な入力を `Record<string, unknown>` に変換 |
| `validateName()` | `name` フィールドの型・長さを手動チェック |
| `validateTitle()` | `title` フィールドの型・長さを手動チェック |

`validateName` や `validateTitle` が内部で実装しているバリデーションルール（最小長・最大長・trim など）は、`schemas.ts` 内の `CreateAppRequestSchema` や `CreateTodoRequestSchema` に **まったく同じルール** が定義されていた。

`http-presenter.ts` でも同様の問題があった。`AppDto` と `TodoDto` は独立したインターフェースとして手書きで宣言されており、`schemas.ts` のスキーマと乖離するリスクがあった。

```typescript
// リファクタリング前（手書き interface）
export type AppDto = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
};
```

---

## 原因と制約

この二重管理が発生した根本原因は、**スキーマが単一情報源として使われていなかった** ことにある。

- `schemas.ts` はバリデーション用スキーマとして定義されていたが、型推論への活用（`z.infer<>`）が後から行われていなかった
- バリデーションヘルパーが先に書かれた後、Zod スキーマが導入されたが、ヘルパーが置き換えられないまま残っていた

DRY（Don't Repeat Yourself）原則への違反であり、将来的に：
- `schemas.ts` のルールを変更してもヘルパー側が追従しない
- `AppDto` の型定義がスキーマと乖離して型エラーが実行時まで気づけない

というリスクを抱えていた。

---

## 解決策

**スキーマを単一情報源（Single Source of Truth）にする。**

1. `request-validation.ts` の手書きヘルパーをすべて削除し、各 parse 関数が `schema.safeParse()` を直接呼ぶ
2. `http-presenter.ts` の `AppDto` / `TodoDto` を `z.infer<typeof AppDtoSchema>` / `z.infer<typeof TodoDtoSchema>` に置き換える

---

## 実装の詳細

### request-validation.ts の変更（122 行 → 53 行）

リファクタリング後の `request-validation.ts` は次の形になった。

```typescript
import { AppError } from '../models/app-error';
import type { CreateAppInput, UpdateAppInput } from '../services/app-usecase';
import type { CreateTodoInput, UpdateTodoInput } from '../services/todo-usecase';
import {
  CreateAppRequestSchema,
  UpdateAppRequestSchema,
  CreateTodoRequestSchema,
  UpdateTodoRequestSchema,
} from './schemas';

function toValidationError(issues: { message: string }[]): AppError {
  return new AppError(
    'VALIDATION_ERROR',
    issues[0]?.message ?? 'Invalid request body',
  );
}

export function parseCreateAppInput(body: unknown): CreateAppInput {
  const result = CreateAppRequestSchema.safeParse(body);
  if (!result.success) throw toValidationError(result.error.issues);
  return result.data;
}

export function parseUpdateAppInput(appId: string, body: unknown): UpdateAppInput {
  const result = UpdateAppRequestSchema.safeParse(body ?? {});
  if (!result.success) throw toValidationError(result.error.issues);
  return { appId, ...result.data };
}

// parseCreateTodoInput / parseUpdateTodoInput も同じパターン
```

ポイントは以下の 3 点。

1. **`toRecord()` の廃止** — Zod の `safeParse()` は `unknown` を直接受け取れるため、`Record<string, unknown>` への変換は不要
2. **`validateName()` / `validateTitle()` の廃止** — スキーマ側に `.trim().min(1).max(100)` などのルールが定義されているため、手動チェックは重複になっていた
3. **薄いエラー変換レイヤー** — `toValidationError()` は Zod の issues を `AppError` に変換するだけの 1 責務に絞った

---

### http-presenter.ts の DTO 型の変更

```typescript
// Before（手書き）
export type AppDto = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
};

// After（スキーマから推論）
export type AppDto = z.infer<typeof AppDtoSchema>;
export type TodoDto = z.infer<typeof TodoDtoSchema>;
```

`AppDtoSchema` / `TodoDtoSchema` は `schemas.ts` で定義済み。

```typescript
// schemas.ts（変更なし）
export const AppDtoSchema = z.object({
  id: z.string(),
  name: z.string(),
  createdAt: z.string(),
  updatedAt: z.string(),
});
```

これにより `schemas.ts` のフィールドを追加・削除すると、`AppDto` の型も自動的に追従する。型定義の同期漏れがコンパイル時に検出できる。

---

## 注意点

- **`safeParse()` は例外を投げない** — `parse()` と違い、失敗時は `result.success === false` として返る。`parseCreateAppInput` が `AppError` を throw するのは `safeParse` ではなく `toValidationError` の呼び出しによるもの
- **`body ?? {}`** — `parseUpdateAppInput` と `parseUpdateTodoInput` では body が `null` / `undefined` の場合に空オブジェクトを渡す。これにより「body なし = 全フィールドが optional」という挙動がスキーマ側の `.optional()` で処理される
- **スキーマとユースケースの型が一致していること** — `result.data` が `CreateAppInput` に直接 return できるのは、スキーマの shape がユースケース入力型と一致しているから。スキーマを変更するときはユースケース型との整合性を確認すること

---

## まとめ

| 観点 | Before | After |
|---|---|---|
| request-validation.ts の行数 | ~122 行 | ~53 行 |
| バリデーションルールの管理 | ヘルパー側とスキーマ側で二重管理 | `schemas.ts` のみ |
| AppDto / TodoDto の型管理 | 手書き interface | `z.infer<>` でスキーマから自動導出 |
| スキーマ変更時の追従 | 手動（漏れリスクあり） | コンパイル時に自動検出 |

Zod スキーマを最初から `z.infer<>` と `safeParse()` の両軸で活用する設計にしておくと、バリデーションと型定義の二重管理が根本から消える。既存のコードベースに Zod スキーマがあるなら、手書きのバリデーションヘルパーや手書き DTO インターフェースは置き換えの候補として見直す価値がある。
