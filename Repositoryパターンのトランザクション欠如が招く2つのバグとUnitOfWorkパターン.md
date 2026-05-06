# Repository パターンのトランザクション欠如が招く 2 つのバグと UnitOfWork パターン

## 対象読者

- Clean Architecture / レイヤードアーキテクチャでバックエンドを実装しているエンジニア
- `Repository` パターンを導入したが、複数リポジトリをまたぐトランザクション管理に迷っているエンジニア
- コードレビューで「アトミックにしてください」と指摘されたことがあるエンジニア

## この記事が扱うこと / 扱わないこと

| | |
|---|---|
| ✅ 扱う | トランザクション境界の欠如が引き起こす具体的なバグ 2 件、および UnitOfWork パターンの概念と設計案 |
| ❌ 扱わない | UnitOfWork の完全な実装コード、TypeORM / Prisma など ORM での扱い |

---

## 背景

このプロジェクトは Clean Architecture に沿って構成されており、ドメイン層に `AppRepository` / `TodoRepository` インターフェースを定義し、インフラ層 (`mysql-app-repository.ts` など) が MySQL 実装を提供しています。

コードレビュー (`review/feat-mysql-integration-tests-ci-2026-04-25.md`) において、以下の P1 所見が 2 件挙がりました。どちらも「複数の DB 書き込みをまたぐトランザクション境界がない」という同一の構造的原因から発生しています。

---

## 問題 1: app 名重複チェックの race condition

### 症状

```ts
// app-interactor.ts — create()
async function create(input: CreateAppInput): Promise<AppEntity> {
  const duplicated = await appRepository.existsActiveByName(input.name); // ① 存在確認
  if (duplicated) throw new AppError('CONFLICT', 'App name already exists');
  // ↑ ここと ↓ の間に別リクエストが割り込める
  await appRepository.save(app); // ② 書き込み
  return app;
}
```

並行リクエストが ① と ② の間に割り込むと、2 つのリクエストが両方「重複なし」と判断して `save()` まで到達します。その結果、同名の App が 2 件作られ、`409 Conflict` を返すべき場面でデータが壊れます。

### 根本原因

`existsActiveByName()` と `save()` が**別々の DB 呼び出し**であり、その間にトランザクション境界がありません。マイグレーション 003 でアプリ名の UNIQUE INDEX が削除されているため、DB レベルの防御もない状態です。

---

## 問題 2: app 削除と todo カスケードの非アトミック性

### 症状

```ts
// app-interactor.ts — remove()
async function remove(input: DeleteAppInput): Promise<AppEntity> {
  const app = await findExistingApp(input.appId);
  const deletedAt = now();
  const deletedApp: AppEntity = { ...app, deletedAt };
  await appRepository.save(deletedApp);                      // ① app をソフトデリート
  const todos = await todoRepository.listActiveByAppId(app.id);
  for (const todo of todos) {
    await todoRepository.save({ ...todo, deletedAt });       // ② todo を順次ソフトデリート
  }
  return deletedApp;
}
```

② の todo ループの途中でエラーが発生した場合、以下の不整合状態が生まれます。

- app は削除済み (`deletedAt` が設定されている)
- 一部の todo はアクティブのまま（孤児データ）
- リトライすると ① の `findExistingApp` が `NOT_FOUND` を返し、修復不能になる

### 根本原因

`appRepository.save()` と `todoRepository.save()` をまたぐ**共通のトランザクション境界が存在しない**ことです。

---

## 共通する構造的な問題

```ts
// repositories/app-repository.ts — 現在のインターフェース
export interface AppRepository {
  save(app: AppEntity): Promise<void>;
  listActive(): Promise<AppEntity[]>;
  findActiveById(id: string): Promise<AppEntity | null>;
  existsActiveByName(name: string, excludeId?: string): Promise<boolean>;
}
```

このインターフェースには、トランザクションを開始・コミット・ロールバックする手段が一切ありません。各メソッドが独立した DB 呼び出しとして設計されているため、複数の操作を「1 つの論理的な作業単位」としてまとめることができません。

2 件の P1 所見はそれぞれ異なる操作（重複チェック、カスケード削除）ですが、**ドメイン層インターフェースにトランザクション抽象がない**という同じ構造的問題を指しています。

---

## 解決策: UnitOfWork パターン

### 概念

UnitOfWork は「1 つのビジネストランザクションに含まれる変更を追跡し、一括でコミット / ロールバックする」パターンです。Martin Fowler の *Patterns of Enterprise Application Architecture* で体系化されています。

Clean Architecture に適用する場合、**ドメイン層のインターフェースとして定義**し、インフラ層が具体的な DB トランザクションとして実装します。これにより、ドメイン層には MySQL の `BEGIN` / `COMMIT` などの知識が漏れ込みません。

### インターフェース設計案

```ts
// domain/unit-of-work.ts（設計案）
export interface UnitOfWork {
  /** コールバック内の処理をひとつのトランザクションで実行する */
  run<T>(work: (ctx: UnitOfWorkContext) => Promise<T>): Promise<T>;
}

export interface UnitOfWorkContext {
  appRepository: AppRepository;
  todoRepository: TodoRepository;
}
```

### 問題 1 の修正イメージ（race condition）

```ts
async function create(input: CreateAppInput): Promise<AppEntity> {
  return uow.run(async (ctx) => {
    // SELECT ... FOR UPDATE でロックを取得しながら確認する
    const duplicated = await ctx.appRepository.existsActiveByNameForUpdate(input.name);
    if (duplicated) throw new AppError('CONFLICT', 'App name already exists');
    await ctx.appRepository.save(app);
    return app;
  }); // ← ここで COMMIT or ROLLBACK
}
```

### 問題 2 の修正イメージ（非アトミック削除）

```ts
async function remove(input: DeleteAppInput): Promise<AppEntity> {
  return uow.run(async (ctx) => {
    const app = await ctx.appRepository.findActiveById(input.appId);
    if (!app) throw new AppError('NOT_FOUND', 'App not found');
    const deletedAt = now();
    const deletedApp: AppEntity = { ...app, deletedAt };
    await ctx.appRepository.save(deletedApp);
    const todos = await ctx.todoRepository.listActiveByAppId(app.id);
    for (const todo of todos) {
      await ctx.todoRepository.save({ ...todo, deletedAt });
    }
    return deletedApp;
  }); // ← todo ループ途中でエラーなら全体 ROLLBACK
}
```

### MySQL 実装のイメージ

```ts
// infrastructure/mysql-unit-of-work.ts（実装案）
export function createMysqlUnitOfWork(pool: Pool): UnitOfWork {
  return {
    async run(work) {
      const conn = await pool.getConnection();
      await conn.beginTransaction();
      try {
        const ctx = {
          appRepository: createMysqlAppRepository(conn),
          todoRepository: createMysqlTodoRepository(conn),
        };
        const result = await work(ctx);
        await conn.commit();
        return result;
      } catch (err) {
        await conn.rollback();
        throw err;
      } finally {
        conn.release();
      }
    },
  };
}
```

---

## 今回の対応: known limitation として記録

UnitOfWork の導入は以下を伴うため、今回の PR スコープ外と判断しました。

- `AppRepository` / `TodoRepository` インターフェースの設計変更
- MySQL 実装クラスの書き直し（接続単位 → トランザクション単位）
- インタラクター全体のテスト更新

コードレビューへの返信では「既知の制限事項（known limitation）として記録し、高並行環境での本番利用前に別 Issue で対応する」と回答しています。低〜中トラフィック環境では実用上の問題は限定的ですが、**高並行環境での本番投入前には必ず対応が必要**です。

---

## 注意点

### UnitOfWork はドメイン層のインターフェースとして定義する

MySQL の `BEGIN` / `COMMIT` といった具体的な知識をドメイン層に持ち込まないことが Clean Architecture の原則です。`UnitOfWork` インターフェース自体は DB に依存しない形で定義し、実装はインフラ層に委ねます。

### SELECT FOR UPDATE を忘れない

race condition の解消には、トランザクション内に入れるだけでなく、**読み取り時に行ロックを取得する** (`SELECT ... FOR UPDATE`) 必要があります。`existsActiveByName()` をトランザクション内で呼び出すだけでは、ロックなしの SELECT はまだ競合しえます。

### インメモリ実装でのテスト

テスト用のインメモリリポジトリでは、UnitOfWork のコールバックをそのまま実行するだけで OK です。トランザクション相当の処理 (rollback) はインメモリ実装では省略できますが、エラー時の副作用をどう扱うかはテスト設計次第で決めてください。

---

## まとめ

| 問題 | 具体的なリスク | 根本原因 |
|------|------------|----------|
| app 名重複の race condition | 同名 App が複数作成される | `existsActiveByName()` → `save()` が別 DB 呼び出し、ロックなし |
| 削除カスケードの非アトミック性 | app 削除後に todo が孤児化、リトライ不能 | `appRepository.save()` と `todoRepository.save()` をまたぐトランザクション境界なし |

どちらも **`AppRepository` / `TodoRepository` インターフェースにトランザクション抽象がない**という同一の構造的原因から生じています。Clean Architecture で複数リポジトリをまたぐ操作をアトミックにするには、ドメイン層に `UnitOfWork` インターフェースを追加し、インフラ層の MySQL 実装でトランザクションとして具体化するのが定石です。

インターフェース変更は影響範囲が大きいため、コードレビューの段階で問題を明確にして「known limitation + 別 Issue」として切り出す判断も現実的な選択肢の一つです。
