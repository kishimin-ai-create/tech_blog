# サイレントプロダクションのバグの修正: MySQL レジストリに接続されたインメモリ ユーザー リポジトリ

**日付:** 2026-05-13
**コミット:** `0546a19`
**範囲:** `backend/` — インフラストラクチャ層、移行

---

## 対象読者

- Hono / Node.js とリポジトリ パターンを使用するバックエンド エンジニア
- サーバー再起動後に謎の 401 やデータ損失に遭遇したエンジニア
- インフラストラクチャ層の修正に TDD を適用することに興味がある人

---

## バグ: 再起動するたびにすべてのユーザーが消去される

認証システムが構築された後、微妙だが重大な配線ミスが運用環境にひっそりと存在していました。

```ts
// backend/src/infrastructure/mysql-registry.ts  (before fix)
import { createInMemoryUserRepository } from './in-memory-repositories';

// ...
const userRepository = createInMemoryUserRepository(); // ← BUG
```

MySQL レジストリ (実稼働サーバーで使用される構成ルート) は、MySQL ベースのリポジトリではなく、**メモリ内のユーザー リポジトリ** を挿入していました。 `App` および `Todo` リポジトリは両方とも実際のデータベースを指しましたが、`UserRepository` は Node.js プロセスの間のみ存在するプレーンな JavaScript `Map` にデータを保存しました。

### 観察される症状

|イベント |結果 |
|---|---|
|ユーザーがサインアップ | ✅ 成功 (メモリ内マップに書き込まれます) |
|サーバーの再起動 (デプロイ、クラッシュ、スケーリング) |地図が破壊されています |
|ユーザーがログインしようとしています | ❌ `401 Unauthorized` — ユーザーが単に存在しません |
|ユーザーが認証されたエンドポイントを使用しようとしています。 ❌ `401 Unauthorized` — トークン検索が失敗します。

開発中に新たに起動されたサーバーではすべてが動作しているように「見えた」ため、この障害は明らかではありませんでした。これは、再起動が日常的に行われる運用環境でのみ確認できるようになりました。

---

## なぜそれが起こったのか

このプロジェクトは **ポートとアダプター (六角形) アーキテクチャ** に従っています。 `UserRepository` インターフェイスはドメイン層に存在します。

```ts
// backend/src/repositories/user-repository.ts
export interface UserRepository {
  findByEmail(email: string): Promise<UserEntity | null>;
  findByToken(token: string): Promise<UserEntity | null>;
  save(user: UserEntity): Promise<void>;
}
```

開発の初期段階で、このインターフェイスのメモリ内実装は、高速な反復とテストのために作成されました。それはまったく合理的です。間違いは、`mysql-registry.ts` (プロダクション コンポジション ルート) がアセンブルされた時点で、`UserRepository` 用の MySQL アダプターがまだ書かれておらず、インメモリ バージョンがプレースホルダーとして使用されていることです。プレースホルダーは決して置き換えられませんでした。

これを明らかにするための実行時エラー、型エラー、テストの失敗はありませんでした。どちらの実装も `UserRepository` を満たすため、TypeScript は沈黙を保っていました。

---

## 修正: TDD 主導の実装

この修正は、実際のデータベースを起動せずに正確性を確保するために、厳格なレッド→グリーンサイクルに従って行われました。

### ステップ 0: 移行 — `users` テーブルの作成

アプリケーション コードを記述する前に、スキーマを定義するために移行ファイルが追加されました。

```sql
-- backend/migrations/005_create_user_table.sql
CREATE TABLE IF NOT EXISTS users (
  id VARCHAR(36) NOT NULL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  token VARCHAR(36) NOT NULL UNIQUE,
  passwordHash VARCHAR(255) NOT NULL,
  createdAt DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

主な決定事項:
- `id` および `token` は `VARCHAR(36)` であり、UUID v4 文字列と一致します。
- `email` と `token` は両方とも `UNIQUE` 制約を保持するため、データベースはアプリケーション ロジックとは関係なく不変条件を強制します。
- `IF NOT EXISTS` により、移行を安全に再実行できるようになります。

### ステップ 1: Kysely `Database` タイプを拡張する

Kysely のタイプ セーフティは、`Database` インターフェイスが完了しているかどうかに依存します。 `users` を追加しない場合、`users` テーブルに対するすべてのクエリは型エラーになります。

```ts
// backend/src/infrastructure/db.ts (added)
export interface UserTable {
  id: string;
  email: string;
  token: string;
  passwordHash: string;
  createdAt: Date;
}

export interface Database {
  App: AppTable;
  Todo: TodoTable;
  users: UserTable;   // ← new
}
```

これは純粋な型アノテーションの変更であり、実行時への影響はありませんが、リポジトリ実装全体でコンパイラチェックされたクエリのロックが解除されます。

### ステップ 2: 🔴 RED — 最初にテストを作成する

まだ存在しない `createMysqlUserRepository` 関数に対して 8 つの単体テストが作成されました。この時点でテスト スイートを実行するとインポート エラーが発生し、正しい RED 状態が確認されました。

テスト戦略では、**手動の Kysely モック** (流暢な Kysely ビルダー API を模倣する `vi.fn()` スタブのチェーン) を使用したため、実際のデータベース接続は必要ありませんでした。

```ts
function makeMockDb() {
  const executeTakeFirst = vi.fn();
  const execute = vi.fn();
  const onDuplicateKeyUpdate = vi.fn(() => ({ execute }));
  const values = vi.fn(() => ({ onDuplicateKeyUpdate }));
  const insertInto = vi.fn(() => ({ values }));
  const where2 = vi.fn(() => ({ executeTakeFirst }));
  const where1 = vi.fn(() => ({ where: where2, executeTakeFirst }));
  const selectAll = vi.fn(() => ({ where: where1 }));
  const selectFrom = vi.fn(() => ({ selectAll }));

  return {
    db: { selectFrom, insertInto } as unknown as Parameters<
      typeof createMysqlUserRepository
    >[0],
    mocks: { /* all fns */ },
  };
}
```

8 つのテスト ケースでは以下がカバーされました。

|方法 |シナリオ |
|---|---|
| `findByEmail` |行が見つからない → `null` を返す |
| `findByEmail` |行が見つかりました → `UserEntity` を返します |
| `findByEmail` | DB がスロー → `AppError('REPOSITORY_ERROR')` として再スロー |
| `findByToken` |行が見つからない → `null` を返す |
| `findByToken` |行が見つかりました → `UserEntity` を返します |
| `findByToken` | DB がスロー → `AppError('REPOSITORY_ERROR')` として再スロー |
| `save` |ハッピー パス → 正しい値で `insertInto('users')` を呼び出します。
| `save` | DB がスロー → `AppError('REPOSITORY_ERROR')` として再スロー |

### ステップ 3: 🟢 緑 — リポジトリを実装する

失敗したテストを実施すると、`mysql-user-repository.ts` が作成されました。

```ts
// backend/src/infrastructure/mysql-user-repository.ts (key excerpts)

function rowToUser(row: { id: string; email: string; token: string;
                          passwordHash: string; createdAt: Date }): UserEntity {
  return { id: row.id, email: row.email, token: row.token, passwordHash: row.passwordHash };
}

export function createMysqlUserRepository(db: Kysely<Database>): UserRepository {
  async function findByEmail(email: string): Promise<UserEntity | null> {
    try {
      const row = await db
        .selectFrom('users')
        .selectAll()
        .where('email', '=', email)
        .executeTakeFirst();
      return row ? rowToUser(row) : null;
    } catch (err) {
      throw new AppError('REPOSITORY_ERROR', 'Repository operation failed', { cause: err });
    }
  }

  async function save(user: UserEntity): Promise<void> {
    try {
      await db
        .insertInto('users')
        .values({ ...user, createdAt: new Date() })
        .onDuplicateKeyUpdate({
          email: sql`VALUES(email)`,
          token: sql`VALUES(token)`,
          passwordHash: sql`VALUES(passwordHash)`,
        })
        .execute();
    } catch (err) {
      throw new AppError('REPOSITORY_ERROR', 'Repository operation failed', { cause: err });
    }
  }

  // findByToken follows the same pattern as findByEmail
  return { findByEmail, findByToken, save };
}
```

注目に値する実装の詳細がいくつかあります。

**`rowToUser` は `createdAt` を削除します** — `UserEntity` は意図的に `createdAt` を公開しません。マッピング関数は、オブジェクトの広がりに依存するのではなく、この境界を明示的にします。これにより、余分な列が暗黙的に組み込まれます。

**`save` は `ON DUPLICATE KEY UPDATE` を使用します** — これは MySQL 固有の upsert です。 `id` は主キーであるため、既存のユーザーの再保存 (ログイン後のトークンの更新など) は、アプリケーション レベルで個別に SELECT、次に INSERT、または UPDATE を決定することなく機能します。

**すべてのエラーは `AppError('REPOSITORY_ERROR')` になります** — ドメイン層は `AppError` コードについてのみ認識します。生のデータベース エラーはログに記録されてからラップされ、インフラストラクチャの問題がドメイン ロジックから排除されます。

### ステップ 4: 実リポジトリを接続する

本番環境のバグを解決する 1 行の修正:

```ts
// backend/src/infrastructure/mysql-registry.ts
- import { createInMemoryUserRepository } from './in-memory-repositories';
+ import { createMysqlUserRepository } from './mysql-user-repository';

  const userRepository = createMysqlUserRepository(db);  // ← was createInMemoryUserRepository()
```

---

## テスト結果

実装後:

```
✓ mysql-user-repository.small.test.ts (8 tests)
✓ Full suite: 89 unit tests passing
✓ TypeScript typecheck: clean
✓ Lint: clean
```

---

## 重要なポイント

### 1. リポジトリパターンでは配線ミスが防げない
`UserRepository` インターフェイスは、インメモリ実装と MySQL 実装の両方で満たされます。 TypeScript は、運用環境でどれが「すべき」であるかを伝えることはできません。それは、構成ルートにエンコードされた人間の判断です。コードレビューと統合テストが安全策です。

### 2. 構成のルーツは精査に値する
`mysql-registry.ts` は、運用環境で実際に実行される内容を決定するファイルです。新しいドメイン コンポーネント (リポジトリなど) は、それが表示されるすべてのレジストリでチェックする必要があります。 「`mysql-registry.ts` のすべてのリポジトリは MySQL でサポートされている」と主張するチェックリストまたは自動テストは、これを即座に検出するでしょう。

### 3. TDD はインフラストラクチャアダプターに適しています
`UserRepository` は明確に定義されたインターフェイスであるため、実装が存在する前に Kysely ビルダー チェーンをモックしてテストを作成するのは簡単でした。モック チェーンは冗長に見えますが、各メソッド呼び出しを正確に制御できます。これは、SQL クエリが正しく構築されていることを確認するときにまさに必要なものです。

### 4. エラーラッピングはユースケースではなくアダプターに属する
生の DB エラーをリポジトリ層で `AppError('REPOSITORY_ERROR')` にラップすると、ユースケースとドメイン ロジックに MySQL 固有の障害モードが発生しなくなります。将来データベースがスワップアウトされた場合でも、エラー コントラクトは変わりません。

---

## まとめ

プレースホルダのメモリ内リポジトリが実稼働 MySQL レジストリに接続されたままになっていたため、サーバーが再起動されるたびにすべてのユーザー データが消失し、以前に登録されたユーザーに対して 401 エラーが発生しました。この修正には、TDD を使用して Kysely がサポートする `UserRepository` 実装を作成することが含まれていました。最初に 8 つの単体テストが作成され、次にそれらを合格させるための実装が作成されました。その後、コンポジション ルートで 1 回のインポート スワップが行われました。 89 個の単体テストからなる完全なテスト スイートは、クリーンなタイプチェックと lint で合格します。
