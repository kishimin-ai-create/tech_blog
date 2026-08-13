# Diary Backend API — Comprehensive Failing Test Suite (Red Phase TDD)

## はじめに

TDD の Red フェーズとして、日記バックエンド API の全テストスイートを実装した。
Hono + TypeScript + Clean Architecture で構成されたバックエンドに対し、
実装がゼロの状態から網羅的な failing テストを書いた記録。

---

## 背景

`backend/src/index.ts` は `"Hello Hono!"` を返すスタブだけが存在し、
エンドポイントも、モデルも、サービスも何もない状態だった。
TDD サイクルに従い、まずテストを書いて失敗させる（Red）ことから始めた。

---

## 作成したテストファイル一覧

| ファイル | 種別 | テスト対象 |
|---|---|---|
| `src/models/diary.small.test.ts` | Small (unit) | `generateContentPreview` 純粋関数 |
| `src/models/user.small.test.ts` | Small (unit) | `hashPassword` / `verifyPassword` |
| `src/services/auth.service.small.test.ts` | Small (unit) | `AuthService` — register / login |
| `src/services/diary.service.small.test.ts` | Small (unit) | `DiaryService` — CRUD + ページネーション |
| `tests/integrations/auth.medium.test.ts` | Medium (integration) | POST /api/auth/register, POST /api/auth/login |
| `tests/integrations/diary.medium.test.ts` | Medium (integration) | GET/POST/PUT/DELETE /api/diaries |
| `tests/integrations/helpers.ts` | ヘルパー | JWT ファクトリ、モックリポジトリファクトリ |

---

## ファイル命名規則

Bun の `bunfig.toml` は `*.small.test.ts` / `*.medium.test.ts` しか拾わない。
`*.test.ts` のみのファイルは **サイレントに無視** されるため、サイズプレフィックスが必須。

```toml
[test]
include = [
  "**/*.small.test.ts",
  "**/*.medium.test.ts",
  "**/*.large.test.ts",
]
```

---

## Small テストの設計方針

### `generateContentPreview` — `diary.small.test.ts`

仕様:
- `\n` → スペースに置換
- 処理後 100 文字超なら先頭 100 文字に切り詰めて `...` を末尾に追加
- 100 文字以下なら `...` なし

境界値テストを重視:

```typescript
test("returns content unchanged when length is exactly 100 chars", () => {
  const content = "a".repeat(100);
  const preview = generateContentPreview(content);
  expect(preview).toBe("a".repeat(100));         // no '...'
  expect(preview.endsWith("...")).toBe(false);
});

test("truncates to 100 chars and appends '...' when content is 101 chars", () => {
  const content = "a".repeat(101);
  const preview = generateContentPreview(content);
  expect(preview).toBe("a".repeat(100) + "...");
});
```

改行後に超過する edge case も:

```typescript
test("replaces newlines before applying the 100-char limit", () => {
  // 50 + \n + 60 = 111 chars after newline→space
  const content = "a".repeat(50) + "\n" + "b".repeat(60);
  const preview = generateContentPreview(content);
  expect(preview).toHaveLength(103); // 100 + '...'
  expect(preview.endsWith("...")).toBe(true);
});
```

### `hashPassword` / `verifyPassword` — `user.small.test.ts`

セキュリティ要件:
- `salt:hash` フォーマットを検証
- 同じパスワードでも毎回異なるハッシュになること（ランダムソルト）
- 正しいパスワードで `true`、誤りで `false`

```typescript
test("produces different hashes for the same plaintext due to a random salt", () => {
  const password = "SamePassword1";
  const hash1 = hashPassword(password);
  const hash2 = hashPassword(password);
  expect(hash1).not.toBe(hash2); // random salt → different each time
});
```

---

## Small テスト — サービス層の設計

### `AuthService` — `auth.service.small.test.ts`

`UserRepository` のインターフェースをテストファイル内でインライン定義し、
実装ファイルの import エラーを区別する。

```typescript
// インライン型定義 — 本番の repositories/user.repository.ts の型を先行定義
interface UserRepositoryForTest {
  findAdmin(): Promise<UserForTest | null>;
  findByEmail(email: string): Promise<UserForTest | null>;
  create(data: CreateUserInputForTest): Promise<{ id: string }>;
}
```

パスワードハッシュは `crypto.scryptSync` を使い、テスト内で直接生成。
`user.ts` が実装済みでなくても、ログインの検証テストが書ける:

```typescript
function createTestPasswordHash(plain: string): string {
  const salt = randomBytes(16).toString("hex");
  const hash = scryptSync(plain, salt, 64).toString("hex");
  return `${salt}:${hash}`;
}
```

**ユーザー列挙攻撃対策テスト**:
メールが存在しない場合と、パスワードが間違っている場合で
401 メッセージが完全一致することを検証する:

```typescript
test("wrong-email and wrong-password 401 messages are identical", async () => {
  // ... 二つのサービスで別々に 401 を発生させ...
  if (hasStatusCode(wrongEmailError) && hasStatusCode(wrongPasswordError)) {
    expect(wrongEmailError.message).toBe(wrongPasswordError.message);
  }
});
```

### `DiaryService` — `diary.service.small.test.ts`

`updateDiary` / `deleteDiary` の 404 テストでは、
サービスが内部で `findById` を呼ぶことを前提として設計:

```typescript
test("throws 404 error when the diary to update does not exist", async () => {
  const diaryRepo = createMockDiaryRepo({
    findById: mock(() => Promise.resolve(null)), // not found
  });
  const service = new DiaryService(diaryRepo);

  let thrownError: unknown;
  try {
    await service.updateDiary("non-existent-uuid", { title: "T", content: "C" });
  } catch (e) { thrownError = e; }

  expect(hasStatusCode(thrownError)).toBe(true);
  if (hasStatusCode(thrownError)) {
    expect(thrownError.statusCode).toBe(404);
  }
});
```

`updateDiary` 成功時は `resolves.toBeUndefined()` で void 戻り値を検証:

```typescript
await expect(
  service.updateDiary(SAMPLE_DIARY.id, { title: "Updated", content: "New" }),
).resolves.toBeUndefined();
```

---

## Medium テストの設計方針

### App ファクトリパターン

統合テストはリポジトリをモックで差し替えられるよう、
`createApp(deps)` ファクトリを仮定して設計した:

```typescript
import { createApp } from "../../src/app"; // まだ存在しない → テストが fail する

const app = createApp({
  userRepo: createMockUserRepo(),
  diaryRepo: createMockDiaryRepo(),
  jwtSecret: TEST_JWT_SECRET,
});
```

Hono の `app.request()` を使えば、HTTP サーバーを立ち上げずに
`fetch` ライクにリクエストできる:

```typescript
const response = await app.request("/api/auth/register", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Admin", email: "admin@example.com", password: "Password123" }),
});
expect(response.status).toBe(201);
```

### JWT ヘルパー

`hono/jwt` の `sign()` でテスト用トークンを生成。
`TEST_JWT_SECRET` を `createApp` と共有することで、
アプリが同じ秘密鍵で検証できる:

```typescript
export async function makeAdminToken(userId = TEST_USER_ID): Promise<string> {
  return sign(
    { sub: userId, role: "admin", exp: Math.floor(Date.now() / 1000) + 3600 },
    TEST_JWT_SECRET,
  );
}
```

### エラーメッセージの完全一致テスト

401/403/404 は仕様で定義されたメッセージを厳密に検証:

```typescript
expect(body.message).toBe("Authentication required."); // 401
expect(body.message).toBe("Access denied.");           // 403
expect(body.message).toBe("Resource not found.");      // 404
```

---

## 失敗確認

```
bun test 2>&1

 0 pass
 6 fail
 6 errors
Ran 6 tests across 6 files. [139.00ms]
```

全テストファイルが `Cannot find module` で fail — これが正しい Red 状態。

| テストファイル | 失敗理由 |
|---|---|
| `diary.small.test.ts` | `./diary` が存在しない |
| `user.small.test.ts` | `./user` が存在しない |
| `auth.service.small.test.ts` | `./auth.service` が存在しない |
| `diary.service.small.test.ts` | `./diary.service` が存在しない |
| `auth.medium.test.ts` | `../../src/app` が存在しない |
| `diary.medium.test.ts` | `../../src/app` が存在しない |

---

## 次のステップ（Green フェーズ）

テストを通すために実装すべきファイル:

```
backend/src/
├── models/
│   ├── diary.ts           ← generateContentPreview
│   ├── user.ts            ← hashPassword, verifyPassword
│   └── errors.ts          ← AppError (statusCode, message)
├── repositories/
│   ├── user.repository.ts ← UserRepository インターフェース
│   └── diary.repository.ts← DiaryRepository インターフェース
├── services/
│   ├── auth.service.ts    ← AuthService (register, login)
│   └── diary.service.ts   ← DiaryService (listDiaries, CRUD)
├── controllers/
│   ├── auth.controller.ts ← POST /api/auth/*
│   └── diary.controller.ts← GET/POST/PUT/DELETE /api/diaries
└── app.ts                 ← createApp(deps) ファクトリ
```

TDD の原則に従い、1 テストずつ Green にしていく。

## まとめ

TDD の原則に従い、1 テストずつ Green にしていく。
