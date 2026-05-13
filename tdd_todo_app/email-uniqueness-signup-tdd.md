# サインアップのメールアドレス重複チェックを TDD で実装する

## 対象読者

- Hono / Node.js バックエンドを開発しているエンジニア
- TDD（Red → Green → Refactor → Review）の具体的な進め方を知りたい方
- インメモリストアを使った統合テストの組み方に興味がある方

---

## 背景：何が問題だったか

`POST /api/v1/auth/signup` は以前、同一メールアドレスで何度でも登録を受け付けていた。レスポンスは常に 201 で、呼び出しのたびに別の `user.id` と `token` が発行されていた。アカウントの概念として成立していない状態だった。

修正後の API 仕様：

| シナリオ | ステータス | エラーコード |
|---|---|---|
| 未登録メール＋正常パスワード | 201 | — |
| 登録済みメール | 409 | `EMAIL_ALREADY_EXISTS` |
| 無効なメール形式 | 422 | `VALIDATION_ERROR` |
| パスワードが 8 文字未満 | 422 | `VALIDATION_ERROR` |

---

## Red フェーズ：先にテストを書く

まず `backend/src/tests/integrations/infrastructure/auth.medium.test.ts` に 2 本のテストを追加した。

### ① 重複登録で 409 を返すこと

```ts
it('409: returns EMAIL_ALREADY_EXISTS when the same email is registered twice', async () => {
  // 1 回目：成功する
  await request('POST', '/api/v1/auth/signup', {
    email: 'duplicate@example.com',
    password: 'password123',
  });

  // 2 回目：同一メールで再登録
  const res = await request('POST', '/api/v1/auth/signup', {
    email: 'duplicate@example.com',
    password: 'password123',
  });

  expect(res.status).toBe(409);

  const json = await res.json();
  expect(json.success).toBe(false);
  expect(json.data).toBeNull();
  expect(json.error.code).toBe('EMAIL_ALREADY_EXISTS');
  expect(json.error.message).toBe('This email address is already registered.');
});
```

### ② ストレージをクリアした後は同一メールで 201 が返ること

アカウント削除後の再登録を想定した分離テスト。`clearStorage()` が `userStore` まで正しく到達することを確かめる意図がある。

```ts
it('201: allows the same email to be registered again after storage is cleared', async () => {
  await request('POST', '/api/v1/auth/signup', {
    email: 'reuse@example.com',
    password: 'password123',
  });

  // 登録済みの状態をリセット
  clearStorage();

  const res = await request('POST', '/api/v1/auth/signup', {
    email: 'reuse@example.com',
    password: 'password123',
  });

  expect(res.status).toBe(201);
});
```

この 2 本はこの時点では当然 FAIL する。

---

## Green フェーズ：最小限の実装でテストを通す

### インメモリストアの選択

データベースへの接続を持ち込まず、`Map<string, UserRecord>` をプロセス内に保持する方針を選んだ。統合テストが外部サービスなしで完結し、`clearStorage()` だけでリセットできる点が決め手だった。

**`hono-app.ts`** に `userStore` を依存として受け取るよう `HonoAppDependencies` を拡張：

```ts
export type UserRecord = { id: string; email: string; token: string };

type HonoAppDependencies = {
  appController: AppController;
  todoController: TodoController;
  userStore?: Map<string, UserRecord>;  // optional にして default を持たせる
};
```

`userStore` を optional にしたのは、既存の呼び出しコードを壊さないためだ。`createHonoApp` 内部でデフォルト値を持つ：

```ts
const { userStore = new Map<string, UserRecord>() } = dependencies;
```

### サインアップハンドラーへの重複チェック追加

```ts
app.post('/api/v1/auth/signup', async c => {
  const parsed = parseAuthCredentials(await readRequestBody(c));

  if (!parsed.success) {
    return c.json(buildValidationErrorBody(parsed.message), 422);
  }

  if (userStore.has(parsed.email)) {          // ← 重複チェック
    return c.json(
      {
        success: false,
        data: null,
        error: {
          code: 'EMAIL_ALREADY_EXISTS',
          message: 'This email address is already registered.',
        },
      },
      409,
    );
  }

  const newUser: UserRecord = {
    id: randomUUID(),
    email: parsed.email,
    token: randomUUID(),
  };
  userStore.set(parsed.email, newUser);       // ← 保存

  return c.json(
    { success: true, data: { token: newUser.token, user: { id: newUser.id, email: newUser.email } } },
    201,
  );
});
```

パターンは単純：`Map.has()` で存在確認 → あれば 409 → なければ `Map.set()` で保存。

### `clearStorage()` に `userStore.clear()` を追加

**`registry.ts`** で `userStore` を生成し、`clearStorage` のクロージャ内に `userStore.clear()` を追加した：

```ts
const userStore = new Map<string, UserRecord>();
const app = createHonoApp({ appController, todoController, userStore });

return {
  app,
  clearStorage: () => {
    storage.clear();
    userStore.clear();   // ← 追加
  },
};
```

これでテスト間の状態汚染が防がれ、② の分離テストが成立する。

---

## Refactor フェーズ：4 つの改善

テストが通った状態で、可読性・保守性の改善を行った。

1. **開き括弧直後の改行を追加**（フォーマット統一）
2. **`parseSignupInput` → `parseAuthCredentials` にリネーム**：同じバリデーション関数をログインでも使うため、名前を汎用化した
3. **`UserRecord` 型を名前付き型として抽出**：`{ id: string; email: string }` のインライン定義を排除
4. **`buildValidationErrorBody` ヘルパーを抽出**：422 レスポンスボディの構築を 1 か所に集約

```ts
function buildValidationErrorBody(message: string) {
  return { data: null, success: false, error: { code: 'VALIDATION_ERROR', message } } as const;
}
```

---

## コードレビューフェーズ：発見された 4 つのバグ

リファクタリング後にコードレビューを実施し、P1/P2 の実装バグが複数発見された。

### P1：メールアドレスの大文字小文字が正規化されていない

`parseAuthCredentials` は `email.trim()` だけで `.toLowerCase()` を適用していなかった。これにより `Alice@example.com` と `alice@example.com` が別キーとして扱われ、**重複チェックが簡単に迂回できた**。

```ts
// 修正前
const normalizedEmail = email.trim();

// 修正後
const normalizedEmail = email.trim().toLowerCase();
```

`userStore` のキー比較（`has()` / `set()` / `get()`）はすべてこの正規化済みの値を使うため、パスワード混在ケースを含め一貫した照合が保証される。

### P1：ログインが `userStore` を参照していなかった

ログインハンドラーはサインアップ前後を問わず、任意の構文的に正しいメール＋パスワードで 200 を返していた。毎回ランダムな `user.id` が生成されており、サインアップで作られたアカウントと紐付いていなかった。

修正後：

```ts
app.post('/api/v1/auth/login', async c => {
  const parsed = parseAuthCredentials(await readRequestBody(c));

  if (!parsed.success) {
    return c.json(buildValidationErrorBody(parsed.message), 422);
  }

  const existingUser = userStore.get(parsed.email);
  if (!existingUser) {
    return c.json(
      {
        success: false,
        data: null,
        error: { code: 'INVALID_CREDENTIALS', message: 'Invalid email or password.' },
      },
      401,
    );
  }

  return c.json(
    { success: true, data: { token: existingUser.token, user: { id: existingUser.id, email: existingUser.email } } },
    200,
  );
});
```

あわせてログインの統合テストにも 401 ケース（未登録メール）を追加した。

### P2：発行したトークンが保存されていなかった

サインアップ時に `randomUUID()` でトークンを生成してレスポンスに含めていたが、`UserRecord` には `{ id, email }` しか保存されていなかった。ログイン時に同じトークンを返せず、将来の Bearer 認証ミドルウェアが検証できなかった。

`UserRecord` に `token` フィールドを追加し、生成したトークンを `userStore.set()` で保存することで、ログイン時も同じトークンを返せるようにした。

### P2：`UserRecord` が export されていなかった

`hono-app.ts` 内の `UserRecord` は export されておらず、`registry.ts` が `Map<string, { id: string; email: string }>` というインライン型を独自に持っていた。フィールドが増えた際に片方だけ更新されてもエラーが出ない状態だった。

`export type UserRecord` にして `registry.ts` で `import type { UserRecord } from './hono-app'` するよう修正した。

### アーキテクチャ上の指摘（今後のフォローアップ）

`backend.rules.md` に定義された Clean Architecture の規則上、`userStore` の操作と重複チェックは本来 `services/` 層（AuthInteractor）に置くべきだという指摘があった。現状では HTTP インフラ層（`hono-app.ts`）が業務ロジックを持っている。

ただしこのリファクタリングは複数層にまたがる変更になるため、スコープを分けて専用の作業として追跡することになった。`userStore` が `HonoAppDependencies` 経由で注入可能な構造になっているので、現時点でも HTTP 境界での統合テストは問題なく動作する。

---

## 最終的なテスト結果

サインアップ関連のテストは 9 本すべてパス。全体では 442 本のテストがすべて通過している。

---

## まとめ

| フェーズ | 作業内容 |
|---|---|
| **Red** | 409 テスト・ストレージクリア後の 201 テストを追加 |
| **Green** | `Map<string, UserRecord>` を導入、`has()` → `set()` で重複チェックを実装 |
| **Refactor** | 関数リネーム・型抽出・ヘルパー抽出・フォーマット修正 |
| **Review** | メール正規化漏れ・ログイン未検証・トークン未保存・型 export 漏れの 4 件を修正 |

今回の実装でとくに重要だったのは次の 3 点：

- **`Map` のキーは正規化してから使う**：`.toLowerCase()` を忘れると大文字小文字の違いで重複が素通りする
- **テスト間の状態分離は `clearStorage()` に集約する**：`userStore.clear()` の追加漏れは、② の分離テストが実際に検証していた
- **コードレビューは動くコードにも必要**：Green フェーズで全テストが通った後でも、P1 バグが 2 件残っていた
