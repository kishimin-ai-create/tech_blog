# MySQLコードレビュー対応でエラー原因伝播とマイグレーション安全性を改善した

## 対象読者

- Node.js + TypeScript でバックエンドを書いているエンジニア
- コードレビューで「エラーを握り潰している」と指摘されたことがある人
- `ES Module` 環境でのファイルパス解決に困ったことがある人

## この記事が扱う範囲

- **扱う**: コードレビュー指摘への対応（エラー原因伝播・DB名バリデーション・マイグレーションパス・HTTPステータスコードのフォールバック）
- **扱わない**: MySQL の基本的な使い方、Hono のルーティング設定

---

## 背景

MySQL 実装（`mysql-app-repository.ts`、`mysql-todo-repository.ts`、`migrate.ts`）の
コードレビューで、UNIQUE INDEX とソフトデリートの競合問題（別記事で扱い済み）以外に
いくつかの問題が指摘された。

コミット `4ac3070` と `8aea581` でそれらの問題が修正された。

---

## 問題1：エラーを握り潰してデバッグ情報が失われる

### 修正前のコード

```typescript
// mysql-app-repository.ts（修正前）
try {
  await pool.execute(`INSERT INTO App ...`);
} catch {
  throw new AppError('REPOSITORY_ERROR', 'Repository operation failed');
}
```

`catch` ブロックで元のエラーオブジェクトを無視していた。
`MySQL2` が返す詳細なエラー情報（SQLステート、エラーコード、メッセージ）が全て捨てられ、
「Repository operation failed」という情報量の少ないメッセージだけが残る。

### 修正後のコード

```typescript
// mysql-app-repository.ts（修正後）
} catch (err: unknown) {
  throw new AppError('REPOSITORY_ERROR', 'Repository operation failed', { cause: err });
}
```

`Error` コンストラクタが受け取る `ErrorOptions` の `cause` プロパティを使うことで、
元のエラーが連鎖する。`AppError` クラスも同様に更新された。

```typescript
// app-error.ts（修正後）
public constructor(code: AppErrorCode, message: string, options?: ErrorOptions) {
  super(message, options);
  this.name = 'AppError';
  this.code = code;
}
```

`ErrorOptions` は Node.js v16.9+ / ES2022 で追加された標準の仕組みで、
`new Error('message', { cause: originalError })` とすることで `error.cause` に元のエラーが入る。

この変更により、ログやスタックトレースで「何が本当に起きたか」を追跡できるようになった。

---

## 問題2：DB名がSQLに直接補間されインジェクションのリスクがあった

### 修正前のコード

```typescript
// migrate.ts（修正前）
const dbName = process.env.DB_DATABASE ?? 'TDDTodoAppDB';

await connection.query(`CREATE DATABASE IF NOT EXISTS ${dbName}`);
```

`DB_DATABASE` 環境変数の値が無検証で SQL に直接補間されていた。
`; DROP DATABASE production; --` のような値を注入できるため、セキュリティ上のリスクがある。

### 修正後のコード

```typescript
// migrate.ts（修正後）
const dbName = process.env.DB_DATABASE ?? 'TDDTodoAppDB';

if (!/^\w+$/.test(dbName)) {
  throw new Error(`Invalid DB_DATABASE value: "${dbName}"`);
}

await connection.query(`CREATE DATABASE IF NOT EXISTS ${dbName}`);
```

`\w+`（英数字とアンダースコアのみ）で DB 名を検証し、不正な値の場合は早期に `throw` する。
データベース名はパラメータバインドができないため、このような明示的なバリデーションが必要。

---

## 問題3：ES Module 環境での `process.cwd()` によるパス解決

### 修正前のコード

```typescript
// migrate.ts（修正前）
const migrationsDir = join(process.cwd(), 'migrations');
```

`process.cwd()` は実行時のカレントディレクトリを返す。
`ts-node` や `tsx` などのランナーによっては、`backend/` 以外のディレクトリから起動されることがあり、
パスが意図したものと異なる可能性がある。

### 修正後のコード

```typescript
// migrate.ts（修正後）
import { fileURLToPath } from 'node:url';

const migrationsDir = fileURLToPath(new URL('../../migrations', import.meta.url));
```

`import.meta.url` はモジュールファイル自体の URL（`file:///path/to/backend/src/infrastructure/migrate.ts`）を返す。
`new URL('../../migrations', import.meta.url)` で相対解決すると、
ファイルの位置を基準とした絶対パスが得られる。

これにより、どのディレクトリから実行しても正しく `backend/migrations/` を指すようになった。
ES Module プロジェクトで `__dirname` の代替として使われる標準的なパターン。

---

## 問題4：HTTPステータスコードのフォールバックが欠落していた

### 修正前のコード

```typescript
// http-presenter.ts（修正前）
function statusForErrorCode(code: AppError['code']): number {
  switch (code) {
    case 'VALIDATION_ERROR':  return 422;
    case 'CONFLICT':          return 409;
    case 'NOT_FOUND':         return 404;
    case 'REPOSITORY_ERROR':  return 500;
    // default がない
  }
}
```

`AppError['code']` は TypeScript の union 型なので、コンパイル時には全ケースを列挙できる。
しかし実行時に予期しないコードが入った場合（型アサーションのミスなど）、
`switch` 文が全ケースにマッチせず関数は `undefined` を返す。
Hono の `context.json(body, status)` に `undefined` が渡ると、型エラーになる。

### 修正後のコード

```typescript
// http-presenter.ts（修正後）
function statusForErrorCode(code: AppError['code']): number {
  switch (code) {
    case 'VALIDATION_ERROR':  return 422;
    case 'CONFLICT':          return 409;
    case 'NOT_FOUND':         return 404;
    case 'REPOSITORY_ERROR':  return 500;
    default:                  return 500;
  }
}
```

`default: return 500` を追加することで、予期しないエラーコードが来た場合でも
500 Internal Server Error を返す安全なフォールバックが機能する。

TypeScript の型システムが「コンパイル時に全ケースを網羅できている」と判断する場合でも、
実行時の安全性のために `default` を書くことは有効だ。

---

## まとめ

コードレビューで見つかったこれらの問題は、一見小さな修正に見えるが、
実際のデプロイ環境では深刻な影響を与えうる。

| 問題 | 修正 | 効果 |
|---|---|---|
| エラーの握り潰し | `{ cause: err }` で連鎖 | デバッグ時に根本原因が追跡できる |
| DB名の無検証補間 | `\w+` バリデーション | 意図しない SQL の実行を防ぐ |
| `process.cwd()` 依存 | `import.meta.url` + `fileURLToPath` | 実行環境に依らず正しいパスを参照 |
| switch の default 欠落 | `default: return 500` | 実行時の予期しない動作を防ぐ |

コードレビューが実装の動作保証だけでなく、セキュリティ・デバッグ容易性・堅牢性の向上にも
貢献している例として記録しておく。
