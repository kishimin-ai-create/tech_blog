# ログイン永続化バグの修正：InMemory リポジトリの誤配線とエラーログの不在

**日付:** 2026-05-13  
**コミット:** `0546a19`（MySQL ユーザーリポジトリ）/ `29a2274`（エラーログ追加）  
**対象ファイル:** `backend/` — infrastructure 層、migrations、hono-app.ts

---

## 対象読者

- Hono / Node.js でバックエンドを構築しているエンジニア
- ポート＆アダプタ（ヘキサゴナル）アーキテクチャを実践しているチーム
- 「ローカルでは動くのに本番で 401 になる」という症状に悩んだことがある人

---

## 問題の全体像

本番環境でサーバーを再起動するたびに、登録済みユーザーが全員ログインできなくなっていた。レスポンスは `401 INVALID_CREDENTIALS`。ユーザーは確かに存在するはずなのに、API はユーザーを「知らない」と返し続けた。

同日、もう一つの問題も見つかった。signup / login の catch ブロックが 500 を返すだけで、サーバーログに何も出力しない実装になっており、**本番で何が起きているか一切わからない**状態だった。

この記事では、この 2 つの修正を順に解説する。

---

## 修正 1：InMemory ユーザーリポジトリの誤配線

### 症状

| タイミング | 結果 |
|---|---|
| ユーザーがサインアップ | ✅ 成功（in-memory Map に書き込まれる） |
| サーバー再起動（デプロイ・クラッシュ・スケールイン） | Map が消滅 |
| 同ユーザーがログイン | ❌ `401 Unauthorized` — ユーザーが存在しない |
| 認証済みエンドポイントへのアクセス | ❌ `401 Unauthorized` — トークン照合失敗 |

開発環境では一切再現しない。サーバーを落とさない限り in-memory Map は生きているからだ。

### 根本原因

`mysql-registry.ts` は本番用の**コンポジションルート**（依存関係を組み立てる唯一の場所）だ。ここで `UserRepository` として `createInMemoryUserRepository()` が差し込まれていた。

```ts
// 修正前 — mysql-registry.ts
import { createInMemoryUserRepository } from './in-memory-repositories';

const userRepository = createInMemoryUserRepository(); // ← バグ
```

`App` と `Todo` のリポジトリは正しく MySQL 実装を使っていたが、`UserRepository` だけが抜け落ちていた。開発初期に in-memory 版をプレースホルダーとして使い始め、MySQL 実装が後回しになったまま本番に出てしまったのが原因だ。

**TypeScript はこれを検出できない。** `UserRepository` インターフェースを両実装が満たしている以上、コンパイラに誤りは見えない。

```ts
// backend/src/repositories/user-repository.ts
export interface UserRepository {
  findByEmail(email: string): Promise<UserEntity | null>;
  findByToken(token: string): Promise<UserEntity | null>;
  save(user: UserEntity): Promise<void>;
}
```

どちらの実装もこのインターフェースを正しく実装しているため、型チェックは常に通る。

### 修正の手順

#### Step 0：マイグレーション追加

まず `users` テーブルを定義するマイグレーションを追加した。

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

設計上の判断点：
- `email` と `token` の両方に `UNIQUE` 制約を付けることで、アプリケーション層に頼らずデータベースレベルで一意性を強制する
- `IF NOT EXISTS` により、マイグレーションを複数回実行しても安全

#### Step 1：Kysely の `Database` 型を拡張

Kysely はクエリの型チェックに `Database` インターフェースを使う。`users` テーブルを追加しないと、`selectFrom('users')` などがすべて型エラーになる。

```ts
// backend/src/infrastructure/db.ts（追加分）
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
  users: UserTable;  // ← 追加
}
```

これはランタイムへの影響はゼロだが、以降のクエリ記述すべてをコンパイラが型チェックできるようになる。

#### Step 2：🔴 RED — テストを先に書く

実装ファイルが存在しない状態でテストを 8 件記述した。この時点ではテストはインポートエラーで失敗する（正しい RED 状態）。

テストは**手書きの Kysely モック**を使い、実際のデータベース接続なしで動作する：

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
    mocks: { selectFrom, selectAll, where1, where2, executeTakeFirst,
             insertInto, values, onDuplicateKeyUpdate, execute },
  };
}
```

Kysely のメソッドチェーン（`selectFrom → selectAll → where → executeTakeFirst`）を `vi.fn()` で再現することで、実際の SQL が正しく構築されているかを DB レスで検証できる。

カバーしたケース：

| メソッド | シナリオ |
|---|---|
| `findByEmail` | 行なし → `null` を返す |
| `findByEmail` | 行あり → `UserEntity` を返す |
| `findByEmail` | DB 例外 → `AppError('REPOSITORY_ERROR')` を投げる |
| `findByToken` | 行なし → `null` を返す |
| `findByToken` | 行あり → `UserEntity` を返す |
| `findByToken` | DB 例外 → `AppError('REPOSITORY_ERROR')` を投げる |
| `save` | 正常系 → `insertInto('users')` が正しい値で呼ばれる |
| `save` | DB 例外 → `AppError('REPOSITORY_ERROR')` を投げる |

#### Step 3：🟢 GREEN — MySQL 実装を書く

```ts
// backend/src/infrastructure/mysql-user-repository.ts（主要部分）

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
      console.error('[mysql-user-repository] findByEmail', err);
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
      console.error('[mysql-user-repository] save', err);
      throw new AppError('REPOSITORY_ERROR', 'Repository operation failed', { cause: err });
    }
  }

  // findByToken は findByEmail と同じパターン
  return { findByEmail, findByToken, save };
}
```

実装上の注目点：

**`rowToUser` で `createdAt` を除外する** — `UserEntity` は `createdAt` を持たない。オブジェクトスプレッドで済ませると列が漏れ出す。明示的なマッピング関数にすることでドメイン層とスキーマの境界を明確にしている。

**`save` は `ON DUPLICATE KEY UPDATE` を使う** — MySQL 固有の upsert 構文。同一 `id` を再 save する場合（例：ログイン後のトークン更新）に SELECT → INSERT or UPDATE の分岐をアプリ側で書かなくて済む。

**エラーは必ず `AppError('REPOSITORY_ERROR')` でラップする** — ドメイン層は MySQL の生エラーを知らなくてよい。インフラ詳細をドメインから隔離したまま、エラーコードは統一されたままになる。

#### Step 4：コンポジションルートを修正

本番バグを直した実際の変更は 1 行のインポート入れ替えだった：

```ts
// backend/src/infrastructure/mysql-registry.ts
- import { createInMemoryUserRepository } from './in-memory-repositories';
+ import { createMysqlUserRepository } from './mysql-user-repository';

  const userRepository = createMysqlUserRepository(db);  // ← 修正完了
```

---

## 修正 2：認証ルートのエラーログ追加

### 症状

signup / login のハンドラは、想定外の例外が発生しても 500 を返すだけでサーバーログに何も出力しなかった。

```ts
// 修正前（概略）
} catch (err) {
  // err は闇に消える
  return c.json({ error: 'INTERNAL_ERROR' }, 500);
}
```

本番で 500 が出ても原因を調べる手段がなく、`REPOSITORY_ERROR` や予期しない例外は完全に不可視だった。

### 修正

```ts
// backend/src/infrastructure/hono-app.ts（修正後）

// signup catch ブロック
} catch (err) {
  if (isAppError(err) && err.code === 'CONFLICT') {
    return c.json({ error: { code: 'EMAIL_ALREADY_EXISTS', ... } }, 409);
  }
  // eslint-disable-next-line no-console
  console.error('[signup] unexpected error:', err);   // ← 追加
  return c.json({ error: { code: 'INTERNAL_ERROR', ... } }, 500);
}

// login catch ブロック
} catch (err) {
  if (isAppError(err) && err.code === 'UNAUTHORIZED') {
    return c.json({ error: { code: 'INVALID_CREDENTIALS', ... } }, 401);
  }
  // eslint-disable-next-line no-console
  console.error('[login] unexpected error:', err);    // ← 追加
  return c.json({ error: { code: 'INTERNAL_ERROR', ... } }, 500);
}
```

変更は小さいが効果は大きい：

- `CONFLICT` / `UNAUTHORIZED` など**想定済み AppError は従来通りに処理**される（ログを出さない）
- それ以外の例外（DB 接続失敗、予期しないランタイムエラー）はスタックトレースごとログに残る
- ログのプレフィックスに `[signup]` / `[login]` を付けることで、複数ルートのログが混在しても発生箇所がすぐ特定できる

---

## 注意点

### TypeScript の型チェックはコンポジションルートの誤りを検出できない

インターフェースを複数の実装で満たす設計（ポート＆アダプタ）は、DI の柔軟性を得る代わりに「誰をどこで使うか」の正しさをコンパイラに保証してもらえない。  
**コンポジションルートは人間がレビューする**か、「MySQL レジストリには MySQL 実装のみ」を検証するインテグレーションテストを用意する必要がある。

### `ON DUPLICATE KEY UPDATE` は MySQL 固有

このプロジェクトは MySQL を前提にしているため問題ないが、PostgreSQL へ移行する場合は `ON CONFLICT DO UPDATE` に書き直す必要がある。Kysely は DB に中立な API を提供しているが、この構文はアダプタ層に残る MySQL 依存点の一つだ。

### エラーログの位置

`console.error` に `eslint-disable-next-line no-console` コメントが付いているのは、このプロジェクトが ESLint で `no-console` ルールを有効にしているためだ。意図的なログ出力であることをコード上で明示している。

---

## まとめ

| 修正 | 内容 | コミット |
|---|---|---|
| MySQL ユーザーリポジトリの実装 | `createInMemoryUserRepository()` を本番 MySQL 実装に置き換え。TDD で 8 テスト先行実装 | `0546a19` |
| 認証ルートのエラーログ追加 | signup / login の catch ブロックに `console.error` を追加し、予期しない 500 エラーを可視化 | `29a2274` |

本番サーバー再起動のたびに全ユーザーデータが消えていた根本原因は、コンポジションルートに in-memory 実装が残り続けていたことだった。TypeScript の型システムはこれを見つけられないため、コンポジションルートの変更は慎重なレビューが必要だという教訓が残る。加えて、エラーを握り潰していた catch ブロックにログを追加することで、同種の問題が再発した際に即座に検知できる体制も整えた。
