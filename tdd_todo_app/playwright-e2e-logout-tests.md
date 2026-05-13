# ログアウト機能の Playwright E2E テスト

**コミット：** `0b72636` — `test(e2e): add Playwright logout E2E tests`
**ファイル：** `frontend/e2e/logout.medium.test.ts`

---

## 対象読者

Playwright E2E テストスイートを書いたり拡張したりしているフロントエンド/フルスタックエンジニアで、認証フローのテストにおける実践的なパターン——特に実際のバックエンドを叩かずに事前認証済み状態を準備する方法——を理解したい方。

---

## この記事で扱う内容

- ログアウトの E2E テストが必要だった理由
- 認証済みユーザーをシミュレートするためにナビゲーション前に `localStorage` をシードする方法
- `page.route()` を使って API レスポンスをスタブする方法
- 2つのテストシナリオとその構造
- テストファイルを読みやすく保つヘルパー関数のパターン

ログアウト機能自体の実装（前のコミットでカバー済み）、Playwright プロジェクトの設定、CI 統合については扱いません。

---

## 背景

Todo App TDD プロジェクトは、`'auth'` キーで `localStorage` に認証状態を永続化するために [jotai](https://jotai.org/) の `atomWithStorage` を使用しています。そのキーが存在した状態でアプリが起動すると、ユーザーは認証済みビュー（アプリ一覧画面）に直接たどり着きます。ユーザーが「ログアウト」をクリックすると、atom がクリアされてルーターが LoginPage にリダイレクトします。

ログアウト機能にはすでにユニットと統合のカバレッジがありましたが、完全なブラウザフロー——ボタンのクリック、リダイレクトのアサート、再ログインでセッションが復元されることの確認——を実行する E2E テストがありませんでした。そのギャップがこの作業の動機です。

---

## 核心的な問題：認証済みユーザーとしてテストを開始する方法

単純なアプローチは、認証済みスタートが必要なすべてのテストでログインフォームを操作することです。デメリットは明白です：すべてのテストがログインフローに結合し、スイートが遅くなり、ログイン API に無関係な問題があると壊れます。

Playwright の [`page.addInitScript()`](https://playwright.dev/docs/api/class-page#page-add-init-script) はこれをエレガントに解決します。`addInitScript` で登録されたスクリプトは、最初のナビゲーションの**前に**ページコンテキスト内で実行されます——つまり React アプリがマウントして読み込む前に `localStorage` に書き込めます。

```typescript
async function seedAuthState(page: Page) {
  await page.addInitScript(() => {
    const authState = {
      token: 'test-token',
      user: { id: 'user-1', email: 'user@example.com' },
    };
    localStorage.setItem('auth', JSON.stringify(authState));
  });
}
```

`'auth'` キーは jotai の `atomWithStorage` が起動時に読み込むものと正確に一致しています。`await page.goto('/')` を呼び出した後、アプリはストレージでトークンとユーザーを見つけて認証済みビューをレンダリングします——ログインフォームの操作は一切不要です。

---

## `page.route()` で API レスポンスをスタブする

認証済みビューでは2つの API エンドポイントが呼び出されます：

| エンドポイント | メソッド | 目的 |
|---|---|---|
| `GET /api/v1/apps` | GET | ユーザーのアプリ一覧を読み込む |
| `POST /api/v1/auth/login` | POST | ログインフォームを送信（再ログインシナリオのみ） |

両方を `page.route()` でスタブして、テストを実際のバックエンドなしで実行できるようにします：

```typescript
async function registerAppsListStub(page: Page) {
  await page.route('**/api/v1/apps', async (route) => {
    if (route.request().method() !== 'GET') {
      await route.continue();
      return;
    }
    await fulfillJson(route, 200, { success: true, data: [] });
  });
}

async function registerLoginStub(page: Page) {
  await page.route('**/api/v1/auth/login', async (route) => {
    if (route.request().method() !== 'POST') {
      await route.continue();
      return;
    }
    await fulfillJson(route, 200, {
      success: true,
      data: {
        token: 'test-token',
        user: { id: 'user-1', email: 'user@example.com' },
      },
    });
  });
}
```

ボイラープレート（`status`、`contentType`、`JSON.stringify`）を一箇所にまとめる共有 `fulfillJson` ヘルパー：

```typescript
async function fulfillJson(route: Route, status: number, body: unknown) {
  await route.fulfill({
    status,
    contentType: 'application/json',
    body: JSON.stringify(body),
  });
}
```

HTTP メソッドのガード（`route.request().method()`）に注目してください。これがないと、URL パターンにマッチするすべてのリクエスト——プリフライトリクエストを含む——が傍受されて誤って fulfill されます。マッチしないメソッドに対して `route.continue()` を呼び出すことで、それらのリクエストは変更されずに通過します。

---

## 2つのテストシナリオ

### 1. ログアウトのハッピーパス

「ログアウト」をクリックするとログインページにリダイレクトされ、ログアウトボタンが消えることを確認します。

```typescript
test(
  'logout happy path: when the logout button is clicked, then the user is redirected to the login page and the logout button disappears',
  async ({ page }) => {
    // Arrange — 認証済みユーザーとして開始
    await seedAuthState(page);
    await registerAppsListStub(page);
    await page.goto('/');
    await expectAuthenticatedView(page);

    // Act — ログアウトボタンをクリック
    await page.getByRole('button', { name: 'ログアウト' }).click();

    // Assert — ログインページが表示され、ログアウトボタンが消える
    await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();
    await expect(page.getByRole('button', { name: 'ログアウト' })).toBeHidden();
  },
);
```

AAA（Arrange / Act / Assert）構造がコメントで明示されています。`expectAuthenticatedView` は共有アサーションヘルパーです（後述）。

### 2. ログアウト後の再ログイン

完全な往復を確認します：ログアウト → ログインフォームを入力 → 認証済みビューが復元されることをアサート。

```typescript
test(
  'logout then re-login: after logging out, the user can log in again and return to the authenticated view',
  async ({ page }) => {
    // Arrange — 認証済みユーザーとして開始してからログアウト
    await seedAuthState(page);
    await registerAppsListStub(page);
    await registerLoginStub(page);
    await page.goto('/');
    await expectAuthenticatedView(page);

    await page.getByRole('button', { name: 'ログアウト' }).click();
    await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();

    // Act — 再度ログイン
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Login' }).click();

    // Assert — 認証済みアプリ一覧ビューに戻る
    await expectAuthenticatedView(page);
    await expect(page.getByText('No apps yet. Create your first app!')).toBeVisible();
  },
);
```

このテストは意図的により複雑です。同一ブラウザセッションでログアウトして再ログインするということは、`localStorage` キーがクリアされてから実際のアプリコードによって再度書き込まれることを意味します——スタブはログインフォームが行う POST リクエストをカバーしていますが、状態管理自体ではありません。

---

## ヘルパー関数

すべてのヘルパーはテストと同じファイル内にあります。1つのテストモジュールのために共有フィクスチャファイルのオーバーヘッドを避けながら、テスト本体を読みやすく保ちます。

| ヘルパー | 役割 |
|---|---|
| `fulfillJson` | JSON コンテントタイプを使った低レベルのルート fulfillment |
| `registerAppsListStub` | `GET /api/v1/apps` → 空のリストをスタブ |
| `registerLoginStub` | `POST /api/v1/auth/login` → 成功レスポンスをスタブ |
| `seedAuthState` | ナビゲーション前に `localStorage` に認証状態をシード |
| `expectAuthenticatedView` | 「Todo App TDD」見出しと「ログアウト」ボタンが表示されていることをアサート |

---

## セレクター戦略：`data-testid` を使わない

すべてのセレクターは Playwright のセマンティッククエリメソッドを使用します：

- `getByRole('heading', { name: 'Todo App TDD' })` — `<h1>` 要素にマッチ
- `getByRole('button', { name: 'ログアウト' })` — アクセシブルラベルでボタンにマッチ
- `getByLabel('Email')` / `getByLabel('Password')` — `<label>` テキストでフォーム入力にマッチ
- `getByText('No apps yet...')` — 表示テキストコンテンツにマッチ

このアプローチにより `data-testid` 属性を本番コンポーネント全体に散らばらせることを避け、テストを内部実装の詳細ではなく UI の可視でアクセシブルな構造に結合させます。

---

## テスト結果

```
2 passed (7.6s)  [chromium]
```

両方のシナリオが Playwright プロジェクトの Chromium ブラウザでパスします。実際のネットワークリクエストがないため実行時間は速い——すべてのバックエンドコールはブラウザを出る前に傍受されます。

---

## まとめ

| 関心事 | アプローチ |
|---|---|
| 認証済み状態の準備 | `page.addInitScript()` でナビゲーション前に `localStorage['auth']` をシード |
| 実際のバックエンドから分離 | `page.route()` で `GET /api/v1/apps` と `POST /api/v1/auth/login` をスタブ |
| テスト構造 | インラインコメントを使った AAA |
| セレクター | ロール / ラベル / テキスト——`data-testid` なし |
| ヘルパーの配置 | 同じテストファイル内に同居 |

重要な洞察は `addInitScript` + `page.route()` の組み合わせです：前者はログインフォームを操作する必要をなくし、後者はライブバックエンドの必要をなくします。合わせることで各テストが高速で、決定論的で、テスト対象の振る舞い——ログアウトフロー自体——に集中できます。

---

## 対象読者

Playwright E2E テスト スイートを作成または拡張しており、認証フローをテストするための実践的なパターン、特に実際のバックエンドにアクセスせずに認証前の状態を調整する方法を理解したいと考えているフロントエンド/フルスタック エンジニア。

---

## この記事の内容

- ログアウトのための E2E テストが必要な理由
- 認証されたユーザーをシミュレートするために、ナビゲーションの前に `localStorage` をシードする方法
- `page.route()` を使用して API 応答をスタブ化する方法
- 2 つのテスト シナリオとその構造
- テスト ファイルを読み取り可能な状態に保つヘルパー関数のパターン

この記事では、ログアウト機能自体の実装 (以前のコミットで説明したもの)、Playwright プロジェクトの構成、または CI 統合については説明しません**。

---

## 背景

Todo App TDD プロジェクトは、[jotai](https://jotai.org/) `atomWithStorage` を使用して、キー `'auth'` の下の `localStorage` に認証状態を保持します。そのキーが存在する状態でアプリが起動すると、ユーザーは認証されたビュー (アプリリスト画面) に直接アクセスします。ユーザーが「ログアウト」をクリックすると、アトムがクリアされ、ルーターは LoginPage にリダイレクトされます。

ログアウト機能にはすでにユニットと統合が含まれていましたが、ボタンをクリックし、リダイレクトをアサートし、再ログインによってセッションが復元されることを確認するという、完全なブラウザ フローを実行する E2E テストはありませんでした。そのギャップがこの作品の原動力となった。

---

## 主な問題: 認証されたユーザーとしてテストを開始する方法

素朴なアプローチは、認証による開始が必要なすべてのテストでログイン フォームを使用することです。欠点は明らかです。すべてのテストがログイン フローに結合され、スイートの速度が低下し、ログイン API に無関係な問題があると中断されます。

Playwright の [`page.addInitScript()`](https://playwright.dev/docs/api/class-page#page-add-init-script) は、これをエレガントに解決します。 `addInitScript` に登録されたスクリプトはすべて、最初のナビゲーションの **前** ページ コンテキスト内で実行されます。つまり、React アプリがマウントして読み取る前に `localStorage` を作成できることになります。

```typescript
async function seedAuthState(page: Page) {
  await page.addInitScript(() => {
    const authState = {
      token: 'test-token',
      user: { id: 'user-1', email: 'user@example.com' },
    };
    localStorage.setItem('auth', JSON.stringify(authState));
  });
}
```

キー `'auth'` は、起動時に jotai の `atomWithStorage` が読み取るものと正確に一致します。 `await page.goto('/')` を呼び出した後、アプリはストレージ内のトークンとユーザーを見つけて、認証されたビューをレンダリングします。ログイン フォームの操作は必要ありません。

---

## `page.route()` を使用した API 応答のスタブ化

認証されたビュー中に 2 つの API エンドポイントが呼び出されます。

|エンドポイント |方法 |目的 |
|---|---|---|
| `GET /api/v1/apps` |入手 |ユーザーのアプリリストをロードする |
| `POST /api/v1/auth/login` |投稿 |ログイン フォームを送信する (再ログイン シナリオのみ) |

どちらも `page.route()` でスタブされているため、テストは実際のバックエンドなしで実行されます。

```typescript
async function registerAppsListStub(page: Page) {
  await page.route('**/api/v1/apps', async (route) => {
    if (route.request().method() !== 'GET') {
      await route.continue();
      return;
    }
    await fulfillJson(route, 200, { success: true, data: [] });
  });
}

async function registerLoginStub(page: Page) {
  await page.route('**/api/v1/auth/login', async (route) => {
    if (route.request().method() !== 'POST') {
      await route.continue();
      return;
    }
    await fulfillJson(route, 200, {
      success: true,
      data: {
        token: 'test-token',
        user: { id: 'user-1', email: 'user@example.com' },
      },
    });
  });
}
```

共有 `fulfillJson` ヘルパーは、ボイラープレート (`status`、`contentType`、`JSON.stringify`) を 1 か所に保持します。

```typescript
async function fulfillJson(route: Route, status: number, body: unknown) {
  await route.fulfill({
    status,
    contentType: 'application/json',
    body: JSON.stringify(body),
  });
}
```

HTTP メソッド (`route.request().method()`) のガードに注意してください。これがないと、URL パターンに一致するすべてのリクエスト (プリフライト リクエストを含む) が傍受され、誤って処理されます。一致しないメソッドに対して `route.continue()` を呼び出すと、それらのリクエストは変更されずに通過します。

---

## 2 つのテスト シナリオ

### 1. ログアウトハッピーパス

Verifies that clicking "ログアウト" redirects to the login page and removes the logout button.

```typescript
test(
  'logout happy path: when the logout button is clicked, then the user is redirected to the login page and the logout button disappears',
  async ({ page }) => {
    // Arrange — start as an authenticated user
    await seedAuthState(page);
    await registerAppsListStub(page);
    await page.goto('/');
    await expectAuthenticatedView(page);

    // Act — click the logout button
    await page.getByRole('button', { name: 'ログアウト' }).click();

    // Assert — login page is shown, logout button is gone
    await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();
    await expect(page.getByRole('button', { name: 'ログアウト' })).toBeHidden();
  },
);
```

AAA（Arrange / Act / Assert）の構造はコメントで明示されています。 `expectAuthenticatedView` は、共有アサーション ヘルパーです (以下を参照)。

### 2.ログアウト後の再ログイン

ログアウト → ログイン フォームに記入 → 認証されたビューが復元されたことをアサートするという完全なラウンドトリップを検証します。

```typescript
test(
  'logout then re-login: after logging out, the user can log in again and return to the authenticated view',
  async ({ page }) => {
    // Arrange — start as an authenticated user, then log out
    await seedAuthState(page);
    await registerAppsListStub(page);
    await registerLoginStub(page);
    await page.goto('/');
    await expectAuthenticatedView(page);

    await page.getByRole('button', { name: 'ログアウト' }).click();
    await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();

    // Act — log in again
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Login' }).click();

    // Assert — back to the authenticated app-list view
    await expectAuthenticatedView(page);
    await expect(page.getByText('No apps yet. Create your first app!')).toBeVisible();
  },
);
```

このテストは意図的により複雑になっています。同じブラウザ セッションでのログアウトと再ログインは、`localStorage` キーがクリアされ、実際のアプリ コードによって再度書き込まれることを意味します。スタブは、状態管理自体ではなく、ログイン フォームが行う POST リクエストをカバーします。

---

## ヘルパー関数

すべてのヘルパーはテストと同じファイル内に存在します。それらを同じ場所に配置すると、テスト本体を読み取り可能にしたまま、単一のテスト モジュールの共有フィクスチャ ファイルのオーバーヘッドが回避されます。

|ヘルパー |役割 |
|---|---|
| `fulfillJson` | JSON コンテンツ タイプを使用した低レベルのルート フルフィルメント |
| `registerAppsListStub` |スタブ `GET /api/v1/apps` → 空のリスト |
| `registerLoginStub` |スタブ `POST /api/v1/auth/login` → 成功応答 |
| `seedAuthState` |ナビゲーション前の認証状態を持つ `localStorage` をシード |
| `expectAuthenticatedView` | Asserts heading "Todo App TDD" and "ログアウト" button are visible |

---

## セレクター戦略: いいえ `data-testid`

すべてのセレクターは、Playwright のセマンティック クエリ メソッドを使用します。

- `getByRole('heading', { name: 'Todo App TDD' })` — `<h1>` 要素と一致します
- `getByRole('button', { name: 'ログアウト' })` — アクセス可能なラベルによってボタンと一致します
- `getByLabel('Email')` / `getByLabel('Password')` — `<label>` テキストによってフォーム入力を照合します。
- `getByText('No apps yet...')` — 表示されているテキスト コンテンツと一致します

このアプローチにより、実稼働コンポーネント間での `data-testid` 属性の分散が回避され、内部実装の詳細ではなく、UI の目に見えるアクセス可能な構造とテストが結合された状態が維持されます。

---

## テスト結果

```
2 passed (7.6s)  [chromium]
```

どちらのシナリオも、Playwright プロジェクトの Chromium ブラウザに対して合格します。実際のネットワーク リクエストがないため、実行時間が高速になります。すべてのバックエンド呼び出しは、ブラウザーを離れる前にインターセプトされます。

---

## まとめ

|懸念事項 |アプローチ |
|---|---|
|認証状態を整える | `page.addInitScript()` はナビゲーション前に `localStorage['auth']` をシード |
|実際のバックエンドから分離 | `page.route()` スタブ `GET /api/v1/apps` および `POST /api/v1/auth/login` |
|テスト構造 |インライン コメント付き AAA |
|セレクター |役割/ラベル/テキスト — いいえ `data-testid` |
|ヘルパーの配置 |同じテスト ファイル内に同じ場所に配置されます。

重要な洞察は、`addInitScript` + `page.route()` の組み合わせです。前者はログイン フォームを使用する必要性を置き換え、後者はライブ バックエンドの必要性を置き換えます。これらを組み合わせることで、各テストが高速かつ決定的になり、テスト対象の動作 (ログアウト フロー自体) に焦点を当てたものになります。
