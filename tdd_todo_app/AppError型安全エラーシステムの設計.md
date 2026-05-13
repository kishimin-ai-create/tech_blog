# AppError 型安全エラーシステムの設計

## 対象読者

- Hono / Express などで TypeScript バックエンドを構築している
- `try-catch` のエラーが `unknown` 型になり、コントローラーでの型ガードを迷っている
- アプリケーションエラーと予期しないエラーを明確に区別したい

---

## 背景

TypeScript では `catch (error)` の `error` は `unknown` 型になる。これをそのまま扱おうとすると、エラーコードや HTTP ステータスへのマッピングが型安全に書けない。

このプロジェクトでは `AppError` クラスを定義し、アプリケーション層で発生するすべての既知エラーをこのクラスで表現する設計を採用した。

---

## AppError の実装

```typescript
// src/models/app-error.ts

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

設計のポイントは3つある。

### 1. `AppErrorCode` を union 型にする

```typescript
export type AppErrorCode =
  | 'VALIDATION_ERROR'
  | 'CONFLICT'
  | 'NOT_FOUND'
  | 'REPOSITORY_ERROR';
```

`string` ではなく union 型にすることで、`switch (code)` が exhaustive になる。新しいエラーコードを追加したとき、対応するステータスマッピングを書き忘れると TypeScript がコンパイルエラーで知らせてくれる。

### 2. `readonly` な `code` プロパティ

```typescript
public readonly code: AppErrorCode;
```

インスタンス生成後に `code` が書き換えられると、エラーハンドリングの判定が壊れる。`readonly` で変更を防ぐ。

### 3. `isAppError()` 型ガード

```typescript
export function isAppError(error: unknown): error is AppError {
  return error instanceof AppError;
}
```

`unknown` 型を受け取り、`AppError` かどうかを判定する型ガード関数。コントローラー層でこれを使うことで、`catch` 節を型安全に書ける。

---

## エラーコードから HTTP ステータスへのマッピング

`http-presenter.ts` で `AppError` を JSON レスポンスに変換する。

```typescript
export function presentError(error: AppError): JsonHttpResponse {
  return {
    status: statusForErrorCode(error.code),
    body: {
      data: null,
      success: false,
      error: {
        code: error.code,
        message: error.message,
      },
    },
  };
}

function statusForErrorCode(code: AppError['code']): number {
  switch (code) {
    case 'VALIDATION_ERROR':  return 422;
    case 'CONFLICT':          return 409;
    case 'NOT_FOUND':         return 404;
    case 'REPOSITORY_ERROR':  return 500;
  }
}
```

`statusForErrorCode` の引数型は `AppError['code']` つまり `AppErrorCode` と等価だ。`switch` のすべての case が union 型のメンバーを網羅しているため、TypeScript は戻り値の型を `number` と確定できる。ここでケースを1つ削除すると、関数の戻り値が `number | undefined` になりコンパイルエラーが出る。

---

## コントローラー層でのエラーハンドリング

`AppController` の各メソッドはすべて `try-catch` で包み、`isAppError()` で判定する。

```typescript
function handleControllerError(error: unknown): JsonHttpResponse {
  if (isAppError(error)) {
    return presentError(error);  // ← 既知のエラーはレスポンスに変換
  }

  throw error;  // ← 未知のエラーは再スロー（バグや想定外の例外）
}
```

`isAppError()` を通過した場合のみ `presentError()` を呼ぶ。それ以外は再スローして、上位の例外ハンドラや監視ツールに委ねる。このパターンによって「既知のアプリケーションエラー」と「バグによる予期しないエラー」を明確に分離できる。

---

## テストで検証していること

### AppError クラス自体のテスト

```typescript
describe('AppError', () => {
  it('sets the name property to "AppError"', () => {
    const error = new AppError('NOT_FOUND', 'App not found');
    expect(error.name).toBe('AppError');
  });

  it('is an instance of Error', () => {
    const error = new AppError('REPOSITORY_ERROR', 'DB failed');
    expect(error).toBeInstanceOf(Error);
  });
});
```

`error.name === 'AppError'` のテストは、`instanceof` 比較が壊れる JavaScript の特殊ケース（古いビルドツールでのクラス継承問題）への備えでもある。

### `isAppError` 型ガードのテスト

```typescript
describe('isAppError', () => {
  it('returns true for an AppError instance', () => {
    expect(isAppError(new AppError('NOT_FOUND', 'x'))).toBe(true);
  });

  it('returns false for a plain Error', () => {
    expect(isAppError(new Error('plain'))).toBe(false);
  });

  it('returns false for a plain object', () => {
    expect(isAppError({ code: 'NOT_FOUND', message: 'x' })).toBe(false);
  });
});
```

`{ code: 'NOT_FOUND', message: 'x' }` は `AppError` の形に似ているが `instanceof` 判定は `false` になる。「形が似ているだけのオブジェクト」を誤検知しないことを明示的に確認している。

### HTTP ステータスマッピングのテスト

```typescript
describe('presentError', () => {
  it('maps VALIDATION_ERROR to status 422', () => {
    expect(presentError(new AppError('VALIDATION_ERROR', 'bad')).status).toBe(422);
  });

  it('maps CONFLICT to status 409', () => {
    expect(presentError(new AppError('CONFLICT', 'dup')).status).toBe(409);
  });

  it('maps NOT_FOUND to status 404', () => {
    expect(presentError(new AppError('NOT_FOUND', 'x')).status).toBe(404);
  });

  it('maps REPOSITORY_ERROR to status 500', () => {
    expect(presentError(new AppError('REPOSITORY_ERROR', 'db')).status).toBe(500);
  });
});
```

エラーコードと HTTP ステータスの対応は仕様として外部に公開されるため、各コードごとに独立したテストケースを設けている。

---

## 層をまたいだエラーの流れ

```
Interactor (AppError をスロー)
  ↓
Controller (isAppError() で判定)
  ↓
HttpPresenter (presentError() でレスポンス化)
  ↓
HonoApp (JSON レスポンスとして返す)
```

各層は `AppError` という共通の型を通じてエラー情報を伝達する。Interactor が HTTP を知る必要はなく、Controller がエラーの内容を再解釈する必要もない。`AppError.code` が「何が起きたか」を表し、Presenter が「それをHTTPでどう表現するか」を担う分離が成立している。

---

## まとめ

- `AppErrorCode` を union 型にすることで、マッピングの網羅性を TypeScript が静的に保証する
- `isAppError()` 型ガードで `unknown` 型の `catch` 節を型安全に扱える
- 既知エラーを `presentError()` で変換し、未知エラーは再スローして分離する
- エラークラス・型ガード・HTTP マッピングそれぞれに独立したテストを書くことで、エラーシステム全体の信頼性を担保できる
