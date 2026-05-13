# コードレビュー指摘5件への対応 — MySQL競合バグ・DTO型・テスト名・CI設定

## 対象読者

- Node.js / TypeScript バックエンドエンジニア
- MySQL を使ったリポジトリ層の実装でアトミック性に不安がある人
- GitHub Actions の Playwright CI を運用している人
- TypeScript の型設計でパブリック API 契約の明示化を検討している人

---

## 背景

フロントエンド CI 整備の PR に対してコードレビュー (`review/frontend-ci-backend-20260428.md`) が実施され、フロントエンドの CI 設定に加えてバックエンドソースにも5件の指摘が上がった。本記事ではその根本原因と修正内容を解説する。

---

## 指摘一覧

| 優先度 | 対象ファイル | 問題 |
|--------|--------------|------|
| P1 | `mysql-app-repository.ts` | 非アトミックな SELECT + INSERT/UPDATE による競合バグ |
| P2 | `ci-nightly.yml` | `@visual` テストが1回のナイトリー実行で2回走る |
| P2 | `hono-app.test.ts` | テスト名と assertion の内容が矛盾 |
| P3 | `http-presenter.ts` | `presentApp` / `presentTodo` の戻り値型が未定義 |
| P3 | `ci-pr.yml` | `--pass-with-no-tests` によるサイレントなカバレッジ消失 |

---

## P1: `mysql-app-repository.ts` — 競合状態バグ

### 問題

修正前の `save()` は次のように実装されていた。

```ts
// 修正前
const [rows] = await pool.execute<ExistsRow[]>(
  'SELECT EXISTS(SELECT 1 FROM App WHERE id = ?) AS `exists`',
  [app.id],
);
if (rows[0].exists === 1) {
  await pool.execute('UPDATE App SET ...', [...]);
} else {
  await pool.execute('INSERT INTO App (...) VALUES (...)', [...]);
}
```

SELECT と INSERT の間に別のリクエストが割り込める。2つのリクエストが同時に `exists = 0` を見た場合、両方が INSERT を実行しようとし、2番目の INSERT が主キー重複エラーで失敗する。このエラーは `AppError('REPOSITORY_ERROR')` としてラップされ HTTP 500 を返す — 正しいセマンティクスではない。

### 根本原因

SELECT + 条件分岐 + INSERT/UPDATE パターンはアトミックではない。MySQL レベルでの排他制御が存在しないため、同時リクエストで必ずこの問題が起きる。

`mysql-todo-repository.ts` は同じ問題を `ON DUPLICATE KEY UPDATE` で解決していたが、`mysql-app-repository.ts` には同じパターンが適用されていなかった。

### 修正

```ts
// 修正後
await pool.execute(
  `INSERT INTO App (id, name, createdAt, updatedAt, deletedAt)
   VALUES (?, ?, ?, ?, ?)
   ON DUPLICATE KEY UPDATE
     name      = VALUES(name),
     updatedAt = VALUES(updatedAt),
     deletedAt = VALUES(deletedAt)`,
  [app.id, app.name, app.createdAt, app.updatedAt, app.deletedAt],
);
```

`INSERT ... ON DUPLICATE KEY UPDATE` は単一の SQL ステートメントとして実行されるためアトミックだ。競合が発生しても MySQL が UPDATE に切り替えるため、アプリケーション層で分岐を持つ必要がない。`mysql-todo-repository.ts` と同じパターンを採用することで一貫性も保たれた。

---

## P2: `ci-nightly.yml` — `@visual` テストの2重実行

### 問題

修正前のナイトリーワークフローでは:

```yaml
# ステップ1: @visual を含む全テストを実行
- run: npx playwright test

# ステップ2: @visual テストを再度実行（重複）
- run: npx playwright test --grep "@visual" --project=chromium
```

全テスト実行ステップに `@visual` テストが含まれており、続く専用ステップでも同じテストが走っていた。CI 時間の無駄に加え、1回目は成功して2回目は失敗するようなフレーキー挙動が混乱を招く可能性があった。

### 修正

```yaml
# 修正後: @visual を除いた全テストを実行
- run: npx playwright test --grep-invert "@visual"
```

`--grep-invert "@visual"` でフル実行ステップから視覚テストを除外し、専用ステップだけで一度実行するようにした。視覚テストは Chromium 専用にすることでフォントレンダリングの差異によるブラウザ間不一致も回避している。

---

## P3: `hono-app.test.ts` — 誤解を招くテスト名

### 問題

```ts
// 修正前
it('PUT with no body falls back to empty body (returns 422 for missing name)', async () => {
  // ...
  expect(res.status).toBe(200); // テスト名と矛盾している
});
```

テスト名は「422 を返す」と書いてあるが、実際の assertion は `200` を期待している。実装は正しく、`name` フィールドは更新リクエストでオプショナルなので 200 が正当だ。しかしテスト名が嘘をついているため、CI ログを見た人が混乱する。

### 修正

```ts
// 修正後
it('PUT with no body is accepted as no-op update (200)', async () => {
```

テスト名を実際の挙動（`name` はオプショナルで、ボディなし PUT は no-op として 200 を返す）に合わせた。テスト名は仕様ドキュメントの一部でもあるため、assertion と矛盾していてはいけない。

---

## P3: `http-presenter.ts` — 明示的な戻り値型

### 問題

```ts
// 修正前
export function presentApp(app: AppEntity) { ... }   // 戻り値型が推論任せ
export function presentTodo(todo: TodoEntity) { ... } // 戻り値型が推論任せ
```

TypeScript の型推論は便利だが、**エクスポートされた関数の戻り値型が未定義だとパブリック API 契約が保護されない**。オブジェクトにプロパティを追加しても TypeScript は警告しないため、意図せず `deletedAt` などの内部フィールドが API レスポンスに漏れてしまう可能性がある。

このプロジェクトのルールでも、全エクスポート関数には明示的な戻り値型を要求している。

### 修正

```ts
// 修正後
export type AppDto = {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
};

export type TodoDto = {
  id: string;
  appId: string;
  title: string;
  completed: boolean;
  createdAt: string;
  updatedAt: string;
};

export function presentApp(app: AppEntity): AppDto { ... }
export function presentTodo(todo: TodoEntity): TodoDto { ... }
```

`AppDto` / `TodoDto` を独立した型として export したことで:

1. 戻り値型が明示されてパブリック API 契約が守られる
2. 呼び出し側がこの型を import して型安全に利用できる
3. orval が生成する OpenAPI スキーマとの整合性チェックが TypeScript レベルで効く

---

## P3: `ci-pr.yml` — `--pass-with-no-tests` の削除

### 問題

```yaml
# 修正前（問題あり）
- run: npx playwright test --grep "@smoke" --pass-with-no-tests --project=chromium
```

`--pass-with-no-tests` は「対象テストが0件でも成功扱い」にするフラグだ。リファクタリングで誤って全 `@smoke` タグを削除しても CI はグリーンのまま通過する。これはサイレントなカバレッジ消失を許容するものだ。

### 修正

フラグを削除し、`e2e/example.spec.ts` に `@smoke` タグを明示した。

```ts
// frontend/e2e/example.spec.ts
test('has title @smoke', async ({ page }) => {
  await page.goto('https://playwright.dev/');
  await expect(page).toHaveTitle(/Playwright/);
});
```

これにより「少なくとも1件の `@smoke` テストが常に存在する」ことが保証される。全タグが誤って消えた場合には CI が明示的に失敗し、問題に気づける。

---

## まとめ

| 修正 | 改善される問題 |
|------|----------------|
| `ON DUPLICATE KEY UPDATE` への置換 | 同時リクエストによる HTTP 500 を排除 |
| `--grep-invert "@visual"` の追加 | ナイトリーでの2重実行と混乱を排除 |
| テスト名の修正 | CI ログの誤解を防ぐ |
| 明示的な DTO 型の追加 | 内部フィールドの API 漏れをコンパイル時に防止 |
| `--pass-with-no-tests` の削除 | 無言のカバレッジ消失を防止 |

これら5件はいずれも「動くが壊れやすい」コードから「型システムと CI がガードとして機能する」コードへの修正だ。バグが発生してから気づくのではなく、開発フロー上の早い段階で問題を検出できるようにする、という方向性で統一されている。
