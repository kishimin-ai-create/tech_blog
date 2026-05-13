# 2026-05-12 作業まとめ: Kysely で SQL 文を型安全なクエリビルダーに移行 & リント・型エラー一括修正

## 対象読者

- TypeScript + Node.js のバックエンド開発者で、生 SQL から型安全なクエリビルダーへの移行に関心がある方
- ESLint / Vitest の特定ルール (`no-misused-promises`, `no-conditional-expect`) の対処法を知りたい方
- Render.com での環境変数管理に関心がある方

## 変更の全体像

2026-05-12 に行ったコミットは以下の 4 件です。

| コミット | 内容 | 規模 |
|---|---|---|
| `f38f2ba` | render.yaml から API URL の値を除去 (`sync: false` に変更) | 1 行 |
| `f78fd83` | `UserProfilePage.medium.test.tsx` の TS2345 修正 | 2 行 |
| `67ab027` | `UserProfilePage` と user-profile テストのリントエラー修正 | 約 5 箇所 |
| `2ca9757` | インライン SQL を Kysely クエリビルダーに置き換え | +154 / −616 行 |

最も規模が大きいのは Kysely 移行です。以降の節ではこの順に解説します。

---

## 1. Kysely で生 SQL をクエリビルダーに移行 (`2ca9757`)

### 背景と動機

バックエンドのリポジトリ実装 (`mysql-app-repository.ts`, `mysql-todo-repository.ts`) はこれまで `mysql2` の `pool.execute()` に生の SQL 文字列を渡す形式でした。この方式には 3 つの課題がありました。

1. **型安全でない**: カラム名をタイプミスしてもコンパイル時に検出できない
2. **読みにくい**: テンプレートリテラルや `?` プレースホルダーが混在する長い文字列
3. **RowDataPacket のキャスト**: `mysql2` の `execute<AppRow[]>()` という型パラメータでキャストが必要で、実際の SELECT カラムとの整合性はコンパイラが保証しない

Kysely はスキーマ生成が不要で、テーブル定義を TypeScript のインターフェースとして書けば、以降のクエリ呼び出しがすべてコンパイル時に検証されます。ORM ほど重くなく、mysql2 をそのまま使いながら型安全なクエリが書ける点が採用理由です。

### `Database` インターフェースのパターン

新設した `backend/src/infrastructure/db.ts` がこの移行の核心です。

```ts
// backend/src/infrastructure/db.ts
import { Kysely, MysqlDialect } from 'kysely';
import { createPool } from 'mysql2';

export interface AppTable {
  id: string;
  name: string;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

export interface TodoTable {
  id: string;
  appId: string;
  title: string;
  completed: number; // MySQL BOOLEAN → 0/1
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

export interface Database {
  App: AppTable;
  Todo: TodoTable;
}

export function createKysely(): Kysely<Database> {
  const config = getMysqlConnectionConfig();
  return new Kysely<Database>({
    dialect: new MysqlDialect({
      pool: createPool({ ...config, timezone: '+00:00', waitForConnections: true, connectionLimit: 10 }),
    }),
  });
}
```

`Database` インターフェースの各プロパティ名がテーブル名に、各インターフェース (`AppTable`, `TodoTable`) のプロパティ名がカラム名に対応します。`.selectFrom('App')` のように書くと、戻り値の型が `AppTable` のフィールドに自動的に絞られます。

### Fluent API への書き換え例

#### `SELECT` + `WHERE`

```ts
// ❌ Before — 生 SQL
const [rows] = await pool.execute<AppRow[]>(
  'SELECT id, name, createdAt, updatedAt, deletedAt FROM App WHERE id = ? AND deletedAt IS NULL',
  [id],
);
const row = rows[0];

// ✅ After — Kysely
const row = await db
  .selectFrom('App')
  .selectAll()
  .where('id', '=', id)
  .where('deletedAt', 'is', null)
  .executeTakeFirst();
```

`executeTakeFirst()` は最初の 1 行だけを返し、なければ `undefined` を返します。以前は `rows[0]` のアクセスを自分で書く必要があったものが、意図が明確な API に置き換わりました。

#### 条件付き `WHERE` の `.$if()` チェーン

```ts
// ❌ Before — if 分岐で 2 本の SQL が必要だった
if (excludeId !== undefined) {
  const [rows] = await pool.execute<ExistsRow[]>(
    'SELECT EXISTS(...WHERE name = ? AND deletedAt IS NULL AND id != ?) AS `exists`',
    [name, excludeId],
  );
  return rows[0].exists === 1;
}
const [rows] = await pool.execute<ExistsRow[]>(
  'SELECT EXISTS(...WHERE name = ? AND deletedAt IS NULL) AS `exists`',
  [name],
);

// ✅ After — .$if() で条件付き WHERE を 1 チェーンで表現
const result = await db
  .selectFrom('App')
  .select(({ fn }) => fn.count<number>('id').as('count'))
  .where('name', '=', name)
  .where('deletedAt', 'is', null)
  .$if(excludeId !== undefined, qb => qb.where('id', '!=', excludeId!))
  .executeTakeFirst();
return Number(result?.count ?? 0) > 0;
```

### `ON DUPLICATE KEY UPDATE` の書き方

MySQL の upsert パターン (`ON DUPLICATE KEY UPDATE`) は Kysely の `.onDuplicateKeyUpdate()` で表現できますが、`VALUES(col)` 関数を使う必要がある箇所では Kysely の `sql` テンプレートタグを組み合わせます。

```ts
// ❌ Before — 生 SQL
await pool.execute(
  `INSERT INTO App (id, name, createdAt, updatedAt, deletedAt)
   VALUES (?, ?, ?, ?, ?)
   ON DUPLICATE KEY UPDATE
     name      = VALUES(name),
     updatedAt = VALUES(updatedAt),
     deletedAt = VALUES(deletedAt)`,
  [app.id, app.name, ...],
);

// ✅ After — Kysely + sql テンプレートタグ
import { sql } from 'kysely';

await db
  .insertInto('App')
  .values({
    id: app.id,
    name: app.name,
    createdAt: new Date(app.createdAt),
    updatedAt: new Date(app.updatedAt),
    deletedAt: app.deletedAt ? new Date(app.deletedAt) : null,
  })
  .onDuplicateKeyUpdate({
    name: sql`VALUES(name)`,
    updatedAt: sql`VALUES(updatedAt)`,
    deletedAt: sql`VALUES(deletedAt)`,
  })
  .execute();
```

`sql\`VALUES(name)\`` は Kysely が理解できるエスケープ済みの生 SQL 断片として扱われます。`onDuplicateKeyUpdate` に値を直接渡すと固定値になってしまうため、INSERT した値を流用したい場合はこの `sql` タグが必須です。

### `mysql-client.ts` の削除

旧来の `createMysqlPool()` を提供していた `mysql-client.ts` は、`db.ts` の `createKysely()` に完全に置き換えられたため削除しました。`mysql-registry.ts` の依存も同様に更新しています。

```ts
// Before
const pool = createMysqlPool();
const appRepository = createMysqlAppRepository(pool);

// After
const db = createKysely();
const appRepository = createMysqlAppRepository(db);
```

この変更の結果、差分は +154 行 / −616 行 で **正味約 460 行の削減** になりました。大部分は `package-lock.json` の整理と、重複していた SQL 文字列・型キャストコードの削除によるものです。

---

## 2. リントエラーの修正 (`67ab027`)

### `import/order`: import の並び順

ESLint の `import/order` ルールは外部パッケージの import をアルファベット順に並べることを要求します。`UserProfilePage.tsx` で `react` が `jotai` より先に書かれていたため違反が発生しました。

```ts
// ❌ Before
import { useState } from 'react'
import { useAtom } from 'jotai'

// ✅ After — j が r より前
import { useAtom } from 'jotai'
import { useState } from 'react'
```

### `no-misused-promises`: `async` 関数を `onSubmit` に直接渡さない

`handleSubmit` は `async` 関数なので `Promise<void>` を返します。React の `onSubmit` は `void` を期待しているため、直接渡すと `@typescript-eslint/no-misused-promises` に引っかかります。

```tsx
// ❌ Before — Promise<void> が onSubmit に渡る
<form onSubmit={handleSubmit}>

// ✅ After — 同期ラッパーで包み .catch() を明示
<form onSubmit={(e) => { handleSubmit(e).catch(() => { /* handled inside */ }) }}>
```

`handleSubmit` の内部には `try/catch` と `setError()` があるので実質的なエラー処理はそこで完結していますが、`.catch()` を明示することで「未処理の reject が外に漏れない」ことをコンパイラと人間の両方に示します。

### `vitest/no-conditional-expect`: `if` ガードを `assert()` に置き換える

判別共用体 (discriminated union) の結果を検証するテストで、`if (result.success)` の中に `expect()` を書くパターンは `vitest/no-conditional-expect` ルール違反になります。条件が `false` のとき `expect()` が無言でスキップされ、テストが誤って「パス」してしまうためです。

```ts
// ❌ Before — if が false なら expect は実行されない
const result = parseUpdateUserProfileInput(body)
if (result.success) {
  expect(result.email).toBe('valid@example.com')
}

// ✅ After — assert() が false なら即テスト失敗し、その後の expect は必ず実行される
import { assert, describe, expect, it } from 'vitest'

const result = parseUpdateUserProfileInput(body)
assert(result.success)                             // false なら TypeError でテスト失敗
expect(result.email).toBe('valid@example.com')    // 型安全かつ常に実行
```

`assert()` は TypeScript の型ガードとしても機能し、以降の行で `result` が成功ブランチに絞り込まれるため、プロパティへのアクセスも型安全になります。同じパターンが 6 箇所あり、すべてこの形式に書き換えました。

### `stylistic/semi`: セミコロン自動補完

backend の ESLint 設定はセミコロン必須 (`stylistic/semi`) です。2 つのテストファイルでセミコロンが抜けていたため、ESLint のオートフィックスで一括修正しました。

```bash
npx eslint --fix backend/src/infrastructure/user-profile-validation.small.test.ts
npx eslint --fix backend/src/tests/user-profile.medium.test.ts
```

173 件の警告が 0 件になりました。ロジックの変更はありません。

### CI nightly: `if: always()` で E2E ステップを確実に実行

`.github/workflows/ci-nightly.yml` で、大規模 E2E テストのステップに `if:` 条件がなかったため、中規模テストが失敗するとそのステップが自動的にスキップされていました。

```yaml
# ❌ Before — 前のステップが失敗するとスキップ
- name: Run large E2E tests
  run: npm run test:e2e:large

# ✅ After — 前のステップの結果に関わらず必ず実行
- name: Run large E2E tests
  if: always()
  run: npm run test:e2e:large
```

Playwright レポートのアップロードステップにはすでに `if: always()` が付いていました。テスト実行ステップも同じ意図で揃えています。

---

## 3. TS2345 の修正: Jotai の型オーバーロード問題 (`f78fd83`)

`UserProfilePage.medium.test.tsx` で `store.set(currentPageAtom, ...)` の第 2 引数の型として `Parameters<typeof store.set>[1]` を使っていましたが、Jotai の `store.set` はオーバーロードが複数あるため TypeScript がこれを `unknown` に解決してしまい TS2345 が発生しました。

```ts
// ❌ Before — Parameters<typeof store.set>[1] が unknown に解決される
store.set(currentPageAtom, 'userProfile' as Parameters<typeof store.set>[1])

// ✅ After — navigation モジュールの Page 型を直接インポートして使う
import type { Page } from '../../../navigation/pages'

store.set(currentPageAtom, 'userProfile' as Page)
```

`Parameters<typeof store.set>[1]` でオーバーロード付き関数の引数型を取ろうとする場合、TypeScript はすべてのオーバーロードのうち「最後の」シグネチャのみを対象とするため、意図した型が取れないことがあります。このケースでは型を明示的にインポートして `as Page` でキャストする方が安全で意図も明確です。

---

## 4. render.yaml のシークレット管理 (`f38f2ba`)

`render.yaml` に API のベース URL (`https://tdd-todo-app-backend.onrender.com`) がハードコードされていました。URL 自体は公開情報ですが、デプロイ設定ファイルに実際の値を埋め込むと後で変更しにくくなるため、`sync: false` に切り替えました。

```yaml
# ❌ Before — 値がリポジトリに露出
envVars:
  - key: VITE_API_BASE_URL
    value: https://tdd-todo-app-backend.onrender.com

# ✅ After — 値は Render ダッシュボードで管理
envVars:
  - key: VITE_API_BASE_URL
    sync: false
```

`sync: false` を指定すると Render は「この環境変数はダッシュボード側で設定してください」と認識し、`render.yaml` 上の値は空になります。URL の変更が必要になった場合もリポジトリへのコミットなしにダッシュボードだけで対応できます。

---

## まとめ

| 変更 | 効果 |
|---|---|
| Kysely 導入 | カラム名・テーブル名のタイポがコンパイル時に検出可能に。+154/−616 行 (正味 −460 行) |
| `sql\`VALUES(col)\`` | Kysely で `ON DUPLICATE KEY UPDATE` の upsert を型安全に表現 |
| `no-misused-promises` → `.catch()` ラッパー | async ハンドラの未処理 reject を防ぐ |
| `no-conditional-expect` → `assert()` | if ガードによるサイレント・パスを排除し、型ガードも兼ねる |
| `if: always()` (CI) | 前のステップが失敗しても大規模 E2E テストが必ず実行される |
| `sync: false` (render.yaml) | 環境変数の実値をリポジトリから切り離し、ダッシュボードで管理 |

### 教訓

- Kysely は「ORMは重い、でも生 SQL は辛い」という中間地点に位置します。スキーマ生成なしで型安全なクエリが書けるため、規模が小さめのプロジェクトでの採用コストが低いです。
- `assert()` は型ガードと `vitest/no-conditional-expect` の両方の問題を同時に解決できます。`if` ガード + `expect` の組み合わせを見つけたらリファクタリングの候補にしてください。
- 環境変数の値はリポジトリに書かず、プラットフォームのシークレット管理機能に任せるのが安全です。`sync: false` はそのための Render 標準機能です。
