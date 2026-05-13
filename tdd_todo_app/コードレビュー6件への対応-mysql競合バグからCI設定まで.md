# コードレビュー6件への対応 — MySQL競合バグからCI設定まで

## 対象読者

- TypeScript / Node.js バックエンドエンジニア
- MySQL を使ったリポジトリ層の実装に興味がある人
- GitHub Actions の Playwright CI を運用している人

## この記事でわかること

`frontend-ci-backend-20260428` レビューで指摘された6件の問題に対して、どのような根本原因があり、どう修正したかを解説します。コミット `8ac31e7` で一括対応しています。

---

## 指摘一覧

| 優先度 | 対象ファイル | 内容 |
|---|---|---|
| P1 | `mysql-app-repository.ts` | 非アトミックな SELECT+INSERT/UPDATE による競合バグ |
| P2 | `ci-nightly.yml` | `@visual` テストが2回実行されていた |
| P2 | `hono-app.test.ts` | テスト名と assertion の内容が矛盾 |
| P3 | `http-presenter.ts` | `presentApp` / `presentTodo` の戻り値型が未定義 |
| P3 | `ci-pr.yml` | `--pass-with-no-tests` によるサイレントな対象消失 |
| P2 | `schemas.ts` / `request-validation.ts` | 二重バリデーション定義（今回は対応見送り） |

---

## P1: `mysql-app-repository.ts` — 競合状態バグ

### 問題

修正前の `save()` は「SELECT で存在確認 → INSERT か UPDATE」という2ステップで動いていました。

```ts
// 修正前
const [rows] = await pool.execute<ExistsRow[]>(
  'SELECT EXISTS(SELECT 1 FROM App WHERE id = ?) AS `exists`',
  [app.id],
);
if (rows[0].exists === 1) {
  await pool.execute('UPDATE App SET name = ?, updatedAt = ?, deletedAt = ? WHERE id = ?', [...]);
} else {
  await pool.execute('INSERT INTO App (id, name, createdAt, updatedAt, deletedAt) VALUES (?, ?, ?, ?, ?)', [...]);
}
```

この2ステップはアトミックではないため、並行リクエストが来ると競合が発生します。

1. リクエストAが `exists = 0` を確認
2. リクエストBも `exists = 0` を確認
3. リクエストAが INSERT 成功
4. リクエストBも INSERT を試みる → **PRIMARY KEY 重複エラー**
5. `catch` ブロックで `AppError('REPOSITORY_ERROR', ...)` に変換 → クライアントに **HTTP 500** が返る

同プロジェクトの `mysql-todo-repository.ts` はすでにアトミック upsert を使っていたため、App側のみ古いパターンが残っていた状態でした。

### 修正

SELECT を撤廃し、単一の `INSERT ... ON DUPLICATE KEY UPDATE` に置き換えます。

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

MySQL が内部的に存在確認と書き込みをアトミックに処理するため、競合状態が根本から消えます。なお、`ExistsRow` 型は `existsActiveByName` で引き続き使用しているため削除しませんでした。

---

## P2: `ci-nightly.yml` — `@visual` テストの2重実行

### 問題

ナイトリー CI には2つのステップが存在していました。

```yaml
# Step 1 — 全テストを実行（@visual も含まれる）
- name: Run Playwright (full suite, all browsers)
  run: npx playwright test

# Step 2 — @visual を再実行（重複）
- name: Run visual regression tests
  run: npx playwright test --grep "@visual" --project=chromium
```

`@visual` テストが1回のナイトリーで2回走っていました。

- CI 時間の無駄
- Step 1 は通過、Step 2 がフレーキーで失敗するなど、混乱しやすい状況になりえる

### 修正

Step 1 に `--grep-invert "@visual"` を追加して、全体実行から `@visual` を除外します。

```yaml
- name: Run Playwright (full suite, all browsers)
  run: npx playwright test --grep-invert "@visual"
```

Step 2（専用の `@visual` ステップ）はそのまま残すため、ビジュアルリグレッションは Chromium 限定で引き続き1回だけ実行されます。

---

## P2: `hono-app.test.ts` — テスト名と assertion の矛盾

### 問題

```ts
// 修正前
it('PUT with no body falls back to empty body (returns 422 for missing name)', async () => {
  ...
  expect(res.status).toBe(200);  // ← 200 を期待しているのに名前は 422
});
```

テスト名は「422 が返る」と言っているのに、実際の assertion は `200` を期待しています。`name` フィールドは PUT で省略可能（no-op update）という正しい仕様がテスト名に反映されていませんでした。

### 修正

```ts
// 修正後
it('PUT with no body is accepted as no-op update (200)', async () => {
```

CI 出力やデバッグ時の混乱を防ぐための修正です。

---

## P3: `http-presenter.ts` — 戻り値型の欠如

### 問題

```ts
// 修正前（推論に頼っていた）
export function presentApp(app: AppEntity) { ... }
export function presentTodo(todo: TodoEntity) { ... }
```

プロジェクトの TypeScript ルール上、public 関数には明示的な戻り値型が必要です。型推論だと、うっかり余分なプロパティを追加してもコンパイルエラーにならず、API 契約が暗黙的に変わるリスクがあります。

### 修正

```ts
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

`AppDto` / `TodoDto` を export することで、他の層がレスポンス型を参照しやすくなります。

---

## P3: `ci-pr.yml` — `--pass-with-no-tests` によるサイレント通過

### 問題

```yaml
# 修正前
run: npx playwright test --grep "@smoke" --pass-with-no-tests --project=chromium
```

`--pass-with-no-tests` フラグは「`@smoke` タグが付いたテストが1件もなくてもパスする」という動作です。テストファイルから `@smoke` タグが全部消えてしまった場合、CI は静かに通過し続けます——スモークテストが実質ゼロになっていることに誰も気づけません。

### 修正

1. `--pass-with-no-tests` を削除
2. `frontend/e2e/example.spec.ts` に `@smoke` タグを追加

```ts
// 修正後
test('has title @smoke', async ({ page }) => {
  await page.goto('https://playwright.dev/');
  ...
});
```

```yaml
# 修正後
run: npx playwright test --grep "@smoke" --project=chromium
```

これにより `@smoke` テストがゼロになった場合、CI が明示的に失敗するようになります。

---

## P2 (対応見送り): `schemas.ts` / `request-validation.ts` の二重バリデーション

### 問題の概要

`schemas.ts`（Zod、OpenAPI ドキュメント用）と `request-validation.ts`（手書きガード、ランタイム用）が同じバリデーション制約（`name`: 1〜100 文字など）を二重に定義しています。どちらかだけ更新された場合、API の実際の動作とドキュメントが乖離します。

### 今回の判断

理想的な修正は `request-validation.ts` で `schema.parse()` を使い、Zod スキーマをランタイムバリデーションに流用することです。ただし:

- Zod のエラー形状を `AppError('VALIDATION_ERROR', ...)` に変換するロジックのテストが必要
- コントローラー層全体の parse 関数更新が伴う
- API レスポンスのエラーメッセージが変わるリスクがある

スコープが広く、今回のフィックスコミットには収まらないため、別タスクとして管理することにしました。

---

## まとめ

| 指摘 | 修正内容 | 効果 |
|---|---|---|
| MySQL 競合バグ | `ON DUPLICATE KEY UPDATE` に統一 | 並行リクエストでの HTTP 500 を排除 |
| @visual 2重実行 | `--grep-invert "@visual"` で除外 | CI 時間の短縮・混乱防止 |
| テスト名の矛盾 | タイトルを実際の挙動に修正 | デバッグ時の可読性向上 |
| 戻り値型なし | `AppDto`/`TodoDto` を追加 | 型安全な API 契約の明文化 |
| `--pass-with-no-tests` | 削除 + `@smoke` タグ追加 | スモークテスト消失の即時検知 |
| 二重バリデーション | 別タスクとして対応見送り | スコープを超える変更を先送り |

今回の対応を通じて、「同じプロジェクト内でも一貫しないパターンが残っていると後から競合バグに発展する」という点が改めて浮き彫りになりました。`mysql-todo-repository.ts` がすでに正しいパターンを持っていたにもかかわらず、`mysql-app-repository.ts` だけ古いままだったのは、レビューなしには見逃しやすいケースです。
