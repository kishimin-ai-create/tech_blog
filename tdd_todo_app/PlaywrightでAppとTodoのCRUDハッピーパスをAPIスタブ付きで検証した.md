# PlaywrightでAppとTodoのCRUDハッピーパスをAPIスタブ付きで検証した

## 対象読者

- Playwright で SPA の主要操作を E2E として検証したい人
- 実バックエンドや DB に依存しない UI ハッピーパステストを書きたい人
- React Query を使う画面で、API mutation と refetch を含む操作を安定して確認したい人

## この記事の範囲

この記事では、`frontend/e2e/crud.spec.ts` に追加した App と Todo の CRUD ハッピーパステストについて扱います。

実バックエンドを起動した完全な end-to-end テストではなく、Playwright の `page.route` で API をインメモリスタブし、UI 操作の流れを検証するテストです。

## 背景

このアプリには App と Todo の操作画面があります。

App には次の操作があります。

- 作成
- 一覧表示
- 詳細表示
- 編集
- 削除

Todo には次の操作があります。

- 作成
- 表示
- 完了状態の切り替え
- 編集
- 削除

これらは個別のコンポーネントテストでは確認できますが、画面遷移、フォーム入力、API mutation 後の再表示、削除確認ダイアログまで含めた一連の流れは Playwright で確認する価値があります。

ただし、E2E を実バックエンドに依存させると、CI で DB や API サーバーを用意する必要があります。今回の目的は UI のハッピーパス検証なので、API は Playwright 側でスタブしました。

## 実装方針

`frontend/e2e/crud.spec.ts` では、`registerApiStub(page)` を用意しています。

この関数は `page.route('**/api/v1/**', ...)` で API request を捕捉し、App と Todo の状態をメモリ上で管理します。

```ts
const apps: AppRecord[] = [];
const todosByAppId = new Map<string, TodoRecord[]>();
```

App ID と Todo ID は連番で生成します。

```ts
let appSequence = 1;
let todoSequence = 1;
```

このスタブが扱う API は、画面が実際に呼び出す `/api/v1/apps` と `/api/v1/apps/:appId/todos` 系の endpoint です。

## 検証している流れ

テスト本体では、ユーザー操作に近い形でハッピーパスを進めています。

### 1. 初期表示

最初に `/` を開き、App が存在しない表示を確認します。

```ts
await expect(page.getByText('No apps yet. Create your first app!')).toBeVisible();
```

### 2. App の作成

`+ Create App` をクリックし、App Name を入力して作成します。

```ts
await page.getByRole('button', { name: '+ Create App' }).click();
await page.getByLabel('App Name').fill('Personal Tasks');
await page.getByRole('button', { name: 'Create' }).click();
```

作成後は一覧へ戻り、作成した App が見えることを確認します。

### 3. App 詳細表示と編集

`View` で詳細へ移動し、`Edit` から App 名を更新します。

```ts
await page.getByLabel('App Name').fill('Work Tasks');
await page.getByRole('button', { name: 'Update' }).click();
```

更新後は詳細画面で新しい名前が表示されることを確認します。

### 4. Todo の作成と完了切り替え

詳細画面で `+ Create Todo` を押し、Todo を作成します。

```ts
await page.getByLabel('Title').fill('Write CRUD E2E test');
await page.getByRole('button', { name: 'Save' }).click();
```

作成された Todo の checkbox をクリックし、完了状態になることも確認します。

```ts
await todoItem.getByRole('checkbox').click();
await expect(todoItem.getByRole('checkbox')).toBeChecked();
```

ここでは `check()` ではなく `click()` を使っています。画面側の checkbox は controlled input で、API 更新後の refetch によって状態が反映されるため、Playwright の `check()` が期待する同期的な状態変化とずれる可能性があるためです。

### 5. Todo の編集と削除

Todo の `Edit` を押してタイトルを更新し、その後 `Delete` と `Confirm` で削除します。

削除後は Todo が非表示になり、空状態のメッセージに戻ることを確認します。

```ts
await expect(page.getByText('Review CRUD E2E test')).toBeHidden();
await expect(page.getByText('No todos yet. Create your first todo!')).toBeVisible();
```

### 6. App の削除

最後に App を削除し、App 一覧の空状態に戻ることを確認します。

```ts
await page.getByRole('button', { name: 'Delete' }).click();
await page.getByRole('button', { name: 'Confirm' }).click();
await expect(page.getByText('No apps yet. Create your first app!')).toBeVisible();
```

## locator の注意点

Todo の `Edit` と App の `Edit` は同じ accessible name を持っています。そのため、単純に `getByRole('button', { name: 'Edit' })` を使うと、意図しないボタンを拾う可能性があります。

今回のテストでは、Todo item の DOM に絞ってからボタンを探しています。

```ts
const todoItem = page
  .locator('div.p-3.border.rounded.bg-white')
  .filter({ hasText: 'Write CRUD E2E test' });
```

これは CSS class に依存しているため、理想的には将来的に `data-testid` やより明確な role/name を追加した方が保守しやすくなります。ただし、現時点では既存 UI を変更せずにテストを安定させるための現実的な選択です。

## smoke test との関係

既存の `frontend/e2e/example.spec.ts` にも `/api/v1/apps` のスタブを追加しています。

これにより、単純な smoke test が Vite proxy 経由で存在しない backend にアクセスし、`ECONNREFUSED` を出す問題を避けています。

## まとめ

Playwright の CRUD ハッピーパステストでは、実 backend を起動せずに UI の主要操作を通しで検証しました。

今回のポイントは次の通りです。

- `page.route` で API をインメモリスタブする
- App と Todo の状態をテスト内で管理する
- controlled checkbox は `click()` 後に `expect(...).toBeChecked()` で待つ
- 同名ボタンは locator scope を絞る

この構成は、UI の基本フローを高速に確認したい場面に向いています。DB を含めた完全な E2E は別レイヤーで扱い、ここでは UI のハッピーパスに責務を絞っています。
