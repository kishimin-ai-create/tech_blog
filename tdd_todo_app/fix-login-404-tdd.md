# `POST /api/v1/auth/login` が 404 を返していたバグを TDD で修正した

## エラー概要

`POST /api/v1/auth/login` を呼び出すと、常に **HTTP 404** が返されていた。

```
POST /api/v1/auth/login
→ 404 Not Found
```

ログイン画面からのリクエストがすべて失敗し、認証フローが機能しない状態だった。

---

## 原因

`backend/src/infrastructure/hono-app.ts` にサインアップ用ルートは登録されていたが、**ログイン用ルートが存在しなかった**。

```ts
// 修正前 — login ルートが無い
app.post('/api/v1/auth/signup', async c => { ... });
// ← /api/v1/auth/login に対応するハンドラーがここに無い
```

Hono はマッチするルートが見つからない場合、デフォルトで 404 を返す。signup と login という 2 つのエンドポイントが必要であることは設計上明らかだったが、実装時に login 側が抜け落ちていた。

---

## TDD サイクルで修正する

### 🔴 RED — 失敗するテストを先に書く

`backend/src/tests/integrations/infrastructure/auth.medium.test.ts` に、ログインエンドポイントを対象にした 3 つのテストを追加した。

```ts
describe('POST /api/v1/auth/login', () => {
  it('200: returns token and user with valid email and password', async () => {
    const res = await request('POST', '/api/v1/auth/login', {
      email: 'test@example.com',
      password: 'password123',
    });

    expect(res.status).toBe(200);

    const json = await res.json() as {
      success: boolean;
      data: { token: string; user: { id: string; email: string } };
    };

    expect(json.success).toBe(true);
    expect(typeof json.data.token).toBe('string');
    expect(UUID_RE.test(json.data.user.id)).toBe(true);
    expect(json.data.user.email).toBe('test@example.com');
  });

  it('422: returns VALIDATION_ERROR when email is invalid', async () => {
    const res = await request('POST', '/api/v1/auth/login', {
      email: 'not-an-email',
      password: 'password123',
    });

    expect(res.status).toBe(422);
    const json = await res.json() as { success: boolean; error: { code: string; message: string } };
    expect(json.success).toBe(false);
    expect(json.error.code).toBe('VALIDATION_ERROR');
  });

  it('422: returns VALIDATION_ERROR when password is less than 8 characters', async () => {
    const res = await request('POST', '/api/v1/auth/login', {
      email: 'test@example.com',
      password: 'short',
    });

    expect(res.status).toBe(422);
    const json = await res.json() as { success: boolean; error: { code: string; message: string } };
    expect(json.success).toBe(false);
    expect(json.error.code).toBe('VALIDATION_ERROR');
  });
});
```

この時点で 3 テストすべてが **404 を受け取って失敗**することを確認した。想定どおりの Red 状態。

### 🟢 GREEN — ルートを追加して通す

`hono-app.ts` に login ルートを追加した。バリデーションロジックは既存の `parseSignupInput` ヘルパーをそのまま流用している。

```ts
app.post('/api/v1/auth/login', async c => {
  const parsed = parseSignupInput(await readRequestBody(c));

  if (!parsed.success) {
    return c.json(
      {
        data: null,
        success: false,
        error: {
          code: 'VALIDATION_ERROR',
          message: parsed.message,
        },
      },
      422,
    );
  }

  return c.json(
    {
      success: true,
      data: {
        token: randomUUID(),
        user: {
          id: randomUUID(),
          email: parsed.email,
        },
      },
    },
    200,  // ← signup の 201 と異なり、login は 200
  );
});
```

**signup との唯一の違いは成功時のステータスコード**だ。新規作成を表す `201 Created` ではなく、既存リソースへの操作を表す `200 OK` を返す。

`parseSignupInput` は以下を検証する：

| 検証項目 | 失敗時のレスポンス |
|---|---|
| `email` が文字列かつ正規表現 `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` にマッチ | 422 + `VALIDATION_ERROR` |
| `password` が 8 文字以上 | 422 + `VALIDATION_ERROR` |

---

## 結果

修正後のレスポンス仕様：

| ケース | ステータス | ボディ |
|---|---|---|
| 有効な email・password | `200 OK` | `{ success: true, data: { token, user: { id, email } } }` |
| email が不正 | `422 Unprocessable Entity` | `{ success: false, error: { code: "VALIDATION_ERROR", message: "..." } }` |
| password が 8 文字未満 | `422 Unprocessable Entity` | `{ success: false, error: { code: "VALIDATION_ERROR", message: "..." } }` |

全 440 テストがパス。型チェック・lint もクリーン。

---

## 補足：現在の実装はプレースホルダー

現時点では `login` も `signup` も**実際のユーザーストアを持たない**。成功時に返す `token` と `user.id` はいずれもリクエストごとに生成されるランダム UUID であり、永続化は行われない。

実際の認証フロー（DB 照合、パスワードハッシュ検証、JWT 発行など）は後続のタスクで実装される予定。今回の修正はあくまで「エンドポイントが存在しないために 404 が返っていた」という構造上の欠落を埋めるものである。

---

## まとめ

- **バグ**: `hono-app.ts` に `POST /api/v1/auth/login` のルート登録が抜けており 404 が返っていた
- **修正**: ルートを追加し、既存の `parseSignupInput` を再利用して email・password バリデーションを適用
- **TDD**: 3 つのテストを Red → Green の順で通すことで修正の正しさをコードで保証した
- **注意点**: 成功時のステータスコードは signup の `201` ではなく `200` — セマンティクス上の使い分けを意識する
