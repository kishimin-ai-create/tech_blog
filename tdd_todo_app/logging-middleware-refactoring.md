# ログミドルウェアのリファクタリング：インラインからメンテナブルな構造へ

## 概要

`backend/src/infrastructure/hono-app.ts` の HTTP リクエスト/レスポンスログミドルウェアをリファクタリングし、コード品質・可読性・保守性を向上させました。複雑なインラインミドルウェアのロジックを、明確な名前を持つ単一責任のヘルパー関数に分離しました。

## 問題

元のログミドルウェアは、複数の関心事が混在した大きなインライン関数として実装されていました：
- リクエストのタイミング計測ロジック
- ルートのフィルタリング
- ステータスコードの確認
- 成功・エラー両方のレスポンスのログ出力
- レスポンスボディからのエラー詳細の抽出

このインラインアプローチは以下の点で困難でした：
- 個々の関心事のテスト
- フローの理解
- リグレッションのリスクなしに変更すること
- コンポーネントの再利用

## 解決策：ヘルパー関数の抽出

リファクタリングにより、それぞれが単一で明確な責任を持つ6つのヘルパー関数を抽出しました：

### 1. **`shouldLogPath(path: string): boolean`**
リクエストパスをログに記録すべきかどうかを判定します。`/api/*` ルートのみログを出力し、`/` や `/doc` などの非 API ルートは除外します。

```typescript
function shouldLogPath(path: string): boolean {
  return path.startsWith('/api/');
}
```

### 2. **`isSuccessStatus(status: number): boolean`**
HTTP ステータスコードが成功レスポンス（2xx 範囲）を表すかどうかを確認します。

```typescript
function isSuccessStatus(status: number): boolean {
  return status >= 200 && status < 300;
}
```

### 3. **`isErrorStatus(status: number): boolean`**
HTTP ステータスコードがエラーレスポンス（4xx または 5xx 範囲）を表すかどうかを確認します。

```typescript
function isErrorStatus(status: number): boolean {
  return status >= 400 && status < 600;
}
```

### 4. **`logSuccessRequest(method, path, status, elapsedTimeMs): void`**
成功した API リクエストを一貫したフォーマットでフォーマットしてログに記録します：
```
[METHOD] path → status (Xms)
```

例：`[GET] /api/v1/apps → 200 (5ms)`

### 5. **`extractErrorDetails(context: Context)`**
不安全な `any` 型キャストを使わず、レスポンスボディからエラーコードとメッセージを安全に抽出します。適切な TypeScript 型ガードを使用：

```typescript
// 型ガード：responseBody が有効なエラーレスポンスオブジェクトかどうかを確認
if (
  typeof responseBody === 'object' &&
  responseBody !== null &&
  'error' in responseBody
) {
  const errorObj = responseBody.error;
  // 型ガード：error オブジェクトが期待される構造を持つかどうかを確認
  if (
    typeof errorObj === 'object' &&
    errorObj !== null &&
    'code' in errorObj &&
    'message' in errorObj &&
    typeof errorObj.code === 'string' &&
    typeof errorObj.message === 'string'
  ) {
    return { code: errorObj.code, message: errorObj.message };
  }
}
```

### 6. **`logErrorRequest(method, path, status, context)`**
エラー詳細を含むフォーマットでエラー API リクエストをフォーマットしてログに記録します：
```
[METHOD] path → ERROR status — code: message
```

例：`[POST] /api/v1/apps → ERROR 409 — CONFLICT: Resource already exists`

エラー詳細が抽出できない場合はグレースフルにフォールバックします：
```
[POST] /api/v1/apps → ERROR 409
```

## 型安全性の改善

**修正前：** コードが不安全な `as any` 型キャストを使用していました：
```typescript
const errorCode = (responseBody as any).error.code;
const errorMessage = (responseBody as any).error.message;
```

**修正後：** コンパイル時の安全性を提供する適切な TypeScript 型ガードに置き換えました：
```typescript
if (
  typeof errorObj === 'object' &&
  errorObj !== null &&
  'code' in errorObj &&
  'message' in errorObj &&
  typeof errorObj.code === 'string' &&
  typeof errorObj.message === 'string'
) {
  return { code: errorObj.code, message: errorObj.message };
}
```

これにより lint エラーが解消され、実行時の安全性と明確さが維持されます。

## ミドルウェアの実装

リファクタリングされたミドルウェアはシンプルで理解しやすくなりました：

```typescript
app.use('*', async (c, next) => {
  const startTime = Date.now();
  await next();

  const path = c.req.path;
  const method = c.req.method;
  const status = c.res.status;
  const elapsedTime = Date.now() - startTime;

  // API ルートのみログを出力（/、/doc などは対象外）
  if (!shouldLogPath(path)) {
    return;
  }

  // レスポンスのステータスに基づいてログを出力
  if (isSuccessStatus(status)) {
    logSuccessRequest(method, path, status, elapsedTime);
  } else if (isErrorStatus(status)) {
    await logErrorRequest(method, path, status, c);
  }
});
```

## 利点

✅ **可読性の向上**：各関数が説明的な名前を持つ単一の明確な目的を持つ  
✅ **テスト容易性の向上**：個々の関数を単独でユニットテストできる  
✅ **型安全性**：不安全な `any` キャストなし；適切な TypeScript 型ガードを全体で使用  
✅ **保守性**：理解・変更・拡張が容易  
✅ **ドキュメント**：各関数を説明する包括的な JSDoc コメント  
✅ **エラーハンドリング**：グレースフルなフォールバックを持つ堅牢なエラー詳細抽出  
✅ **ログの明確さ**：成功とエラーケースの一貫したログフォーマット  

## テスト結果

- **51テストがパス** — すべての新規エラーログテストが正常にパス
- **TypeScript コンパイル**：リファクタリングされたコードでエラーなしにクリーン
- **Linting**：リファクタリングされたミドルウェアで違反なし

## コミット

```
refactor: extract logging middleware helper functions for improved code quality

- Extract shouldLogPath() to determine if a request path should be logged
- Extract isSuccessStatus() to check for 2xx response codes
- Extract isErrorStatus() to check for 4xx-5xx error responses
- Extract logSuccessRequest() for success response logging
- Extract logErrorRequest() for error response logging  
- Extract extractErrorDetails() to safely extract error info using type guards
- Remove unsafe 'any' type casts, replace with proper type guards
- Add comprehensive JSDoc comments
```

## 結論

このリファクタリングは、外部の振る舞いを変えることなく関心事の体系的な抽出を通じてコード品質を向上させる方法を示しています。ミドルウェアは単一責任の原則に従い、完全な型安全性を持ち、成功・エラー両方の API レスポンスの明確なログを提供するようになりました。

## 問題提起

元のロギング ミドルウェアは、複雑な懸念を伴う大規模なインライン関数として実装されました。
- リクエストタイミングロジック
- ルートフィルタリング
- ステータスコードのチェック
- 成功応答とエラー応答の両方の応答ロギング
- レスポンスボディからのエラー詳細の抽出

このインライン アプローチにより、コードでは次のことが困難になりました。
- 個々の懸念事項をテストする
- 流れを理解する
- 回帰の危険を冒さずに変更する
- コンポーネントを再利用する

## 解決策: ヘルパー関数の抽出

リファクタリングにより 6 つのヘルパー関数が抽出され、それぞれに単一の明確な責任があります。

### 1. **`shouldLogPath(path: string): boolean`**
リクエストパスをログに記録するかどうかを決定します。 `/` や `/doc` などの非 API ルートを除き、`/api/*` ルートのみをログに記録します。

```typescript
function shouldLogPath(path: string): boolean {
  return path.startsWith('/api/');
}
```

### 2. **`isSuccessStatus(status: number): boolean`**
HTTP ステータス コードが成功応答 (2xx 範囲) を表しているかどうかを確認します。

```typescript
function isSuccessStatus(status: number): boolean {
  return status >= 200 && status < 300;
}
```

### 3. **`isErrorStatus(status: number): boolean`**
HTTP ステータス コードがエラー応答 (4xx または 5xx の範囲) を表しているかどうかを確認します。

```typescript
function isErrorStatus(status: number): boolean {
  return status >= 400 && status < 600;
}
```

### 4. **`logSuccessRequest(method, path, status, elapsedTimeMs): void`**
成功した API リクエストを一貫した形式でフォーマットしてログに記録します。
```
[METHOD] path → status (Xms)
```

例: `[GET] /api/v1/apps → 200 (5ms)`

### 5. **`extractErrorDetails(context: Context)`**
安全でない `any` 型キャストを使用せずに、応答本文からエラー コードとメッセージを安全に抽出します。適切な TypeScript タイプ ガードを使用します。

```typescript
// Type guard: check if responseBody is a valid error response object
if (
  typeof responseBody === 'object' &&
  responseBody !== null &&
  'error' in responseBody
) {
  const errorObj = responseBody.error;
  // Type guard: check if error object has the expected structure
  if (
    typeof errorObj === 'object' &&
    errorObj !== null &&
    'code' in errorObj &&
    'message' in errorObj &&
    typeof errorObj.code === 'string' &&
    typeof errorObj.message === 'string'
  ) {
    return { code: errorObj.code, message: errorObj.message };
  }
}
```

### 6. **`logErrorRequest(method, path, status, context)`**
エラーの詳細を含む形式を使用して、エラー API リクエストをフォーマットしてログに記録します。
```
[METHOD] path → ERROR status — code: message
```

例: `[POST] /api/v1/apps → ERROR 409 — CONFLICT: Resource already exists`

エラーの詳細を抽出できない場合は正常にフォールバックします。
```
[POST] /api/v1/apps → ERROR 409
```

## タイプの安全性の向上

**変更前:** コードでは安全でない `as any` 型キャストが使用されています。
```typescript
const errorCode = (responseBody as any).error.code;
const errorMessage = (responseBody as any).error.message;
```

**変更後:** コンパイル時の安全性を提供する適切な TypeScript タイプ ガードに置き換えられました。
```typescript
if (
  typeof errorObj === 'object' &&
  errorObj !== null &&
  'code' in errorObj &&
  'message' in errorObj &&
  typeof errorObj.code === 'string' &&
  typeof errorObj.message === 'string'
) {
  return { code: errorObj.code, message: errorObj.message };
}
```

これにより、実行時の安全性と明瞭さを維持しながら、lint エラーが排除されます。

## ミドルウェアの実装

リファクタリングされたミドルウェアはクリーンで理解しやすくなりました。

```typescript
app.use('*', async (c, next) => {
  const startTime = Date.now();
  await next();

  const path = c.req.path;
  const method = c.req.method;
  const status = c.res.status;
  const elapsedTime = Date.now() - startTime;

  // Only log API routes (not /, /doc, etc)
  if (!shouldLogPath(path)) {
    return;
  }

  // Log based on response status
  if (isSuccessStatus(status)) {
    logSuccessRequest(method, path, status, elapsedTime);
  } else if (isErrorStatus(status)) {
    await logErrorRequest(method, path, status, c);
  }
});
```

## 利点

✅ **可読性の向上**: 各関数には、わかりやすい名前が付いた明確な単一の目的があります。
✅ **テスト容易性の向上**: 個々の機能を個別に単体テストできます。
✅ **タイプ セーフティ**: 安全でない `any` キャストはありません。全体を通して使用される適切な TypeScript タイプ ガード
✅ **保守性**: 理解、変更、拡張が容易になります。
✅ **ドキュメント**: 各関数を説明する包括的な JSDoc コメント
✅ **エラー処理**: 適切なフォールバックによる堅牢なエラー詳細抽出
✅ **ログの明瞭さ**: 成功ケースとエラーケースの一貫したログ形式

## テスト結果

- **51 件のテストに合格** - すべての新しいエラー ログ テストに合格しました
- **TypeScript のコンパイル**: リファクタリングされたコードにエラーがなくクリーンです
- **リンティング**: リファクタリングされたミドルウェアに違反はありません

## 専念

```
refactor: extract logging middleware helper functions for improved code quality

- Extract shouldLogPath() to determine if a request path should be logged
- Extract isSuccessStatus() to check for 2xx response codes
- Extract isErrorStatus() to check for 4xx-5xx error responses
- Extract logSuccessRequest() for success response logging
- Extract logErrorRequest() for error response logging  
- Extract extractErrorDetails() to safely extract error info using type guards
- Remove unsafe 'any' type casts, replace with proper type guards
- Add comprehensive JSDoc comments
```

## 結論

このリファクタリングは、外部の動作を変更せずに懸念事項を体系的に抽出することでコードの品質を向上させる方法を示します。このミドルウェアは単一責任の原則に従い、完全にタイプセーフであり、成功とエラーの両方の API 応答の明確なログを提供します。
