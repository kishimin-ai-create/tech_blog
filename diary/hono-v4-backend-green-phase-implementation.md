# Hono v4 バックエンド実装 — TDD Green フェーズ完走記録

## 概要

TDD の Red フェーズで書かれた 82 本のテスト（Small 4 ファイル × unit + Medium 2 ファイル × integration）をすべて通す実装を行った。  
フレームワークは Hono v4、ランタイムは Bun、バリデーションは Zod v4。  
途中で Hono v4 と Zod v4 特有のハマりポイントを 2 つ踏んだので、それも記録する。

---

## 作成したファイル一覧

| ファイル | 役割 |
|---|---|
| `src/models/diary.ts` | `generateContentPreview` — 改行→スペース変換・100文字切り詰め |
| `src/models/user.ts` | `hashPassword` / `verifyPassword` — scrypt + randomBytes |
| `src/repositories/user.repository.ts` | `IUserRepository` インターフェース |
| `src/repositories/diary.repository.ts` | `IDiaryRepository` インターフェース |
| `src/services/auth.service.ts` | `AuthService` — register(409) / login(JWT) |
| `src/services/diary.service.ts` | `DiaryService` — CRUD + 404 スロー |
| `src/controllers/auth.controller.ts` | `POST /register` / `POST /login` with Zod validation |
| `src/controllers/diary.controller.ts` | Diary CRUD + JWT auth middleware + admin check |
| `src/app.ts` | `createApp(deps)` — DI対応のHonoアプリファクトリ |

---

## 実装のポイント

### `generateContentPreview`

テストが求める仕様は明快だった：

```typescript
export function generateContentPreview(content: string): string {
  const processed = content.replace(/\n/g, " ");
  if (processed.length <= 100) return processed;
  return processed.slice(0, 100) + "...";
}
```

改行を先にスペース変換してから文字数を測ること。順序を逆にすると境界値テストが壊れる。

### `hashPassword` / `verifyPassword`

Node.js の `scryptSync` + `randomBytes` を使う。`verifyPassword` で `timingSafeEqual` を呼ぶ前に **バッファ長チェック** を入れるのが重要。テスト上では意図的に `"placeholder:hash"` のような不正な stored 値が渡されるケースがあり、長さ不一致で `timingSafeEqual` が throw するのを try/catch で拾う必要がある。

```typescript
if (actualHash.length !== expectedBuf.length) return false;
return timingSafeEqual(actualHash, expectedBuf);
```

### AuthService — JWT の署名

Hono の `sign` を使って JWT を生成。ペイロードは `{ sub, role, exp }` の 3 フィールド。

```typescript
const accessToken = await sign(
  { sub: user.id, role: user.role, exp: Math.floor(Date.now() / 1000) + 3600 },
  this.config.jwtSecret,
);
```

### createApp ファクトリ

テスト用のモックリポジトリを注入できるよう DI パターンを採用：

```typescript
export function createApp(deps: {
  userRepo: IUserRepository;
  diaryRepo: IDiaryRepository;
  jwtSecret: string;
}): Hono { ... }
```

これにより `app.request()` を使った integration テストで本物の DB を使わずに済む。

---

## ハマりポイント 1 — Hono v4 の `verify()` はアルゴリズム指定が必須

Hono v4.12.x では `hono/jwt` の `verify()` 関数がアルゴリズムを必須引数として要求するようになっている。

```
JwtAlgorithmRequired: JWT verification requires "alg" option to be specified
```

`sign()` はデフォルト HS256 で動くが、`verify()` は明示指定が必要：

```typescript
// NG: verify(token, jwtSecret)
// OK:
const payload = await verify(token, jwtSecret, "HS256");
```

integration テストで valid な admin token を送っても全部 401 が返ってきてしばらく迷った。

---

## ハマりポイント 2 — Zod v4 の `z.string().uuid()` は RFC 4122 strict

Zod v4 の `z.string().uuid()` は UUID のバージョンビット（位置 14 が `4`、位置 19 が `8/9/a/b`）を厳格に検証する。

テストで使われている `UNKNOWN_UUID = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"` はフォーマットとしては UUID 形式だが、バージョンビットが無効なため Zod に reject される。

結果、「Not Found → 404」のテストが「Invalid ID → 400」になってしまう。

解決策：UUID バリデーションを lenient な hex regex に変更する。

```typescript
// Zod v4 strict — NG for non-v4 UUIDs
z.string().uuid()

// Lenient hex format — OK
const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
z.string().regex(uuidRegex)
```

「invalid UUID format」のテスト（`"not-a-uuid"` → 400）と「valid-format but not found」のテスト（`"aaaaaaaa-..."` → 404）を両立させるには、フォーマット検証のみ行い、RFC 4122 バージョン検証は行わない正規表現を使う必要がある。

---

## `auth.medium.test.ts` のプレースホルダーハッシュを更新

login happy path テストには `passwordHash: "placeholder:hash"` というコメント付きのプレースホルダーが入っていた。Green フェーズで実際の scrypt ハッシュに置き換えるよう指示されていた。

```bash
bun -e "
import { scryptSync, randomBytes } from 'crypto'
const salt = randomBytes(16).toString('hex')
const hash = scryptSync('Password123', salt, 64).toString('hex')
console.log(salt + ':' + hash)
"
```

生成したハッシュをテストに埋め込み、login の 200 OK テストが通るようにした。

---

## TypeScript 型エラーの解決

`bun run typecheck`（実態は `tsgo --noEmit`）で 2 種類の型エラーが残った：

1. **`crypto` / `Buffer` が見つからない** — `tsgo` が `@types/bun` を自動検出しない  
2. **`bun:test` モジュールが見つからない** — 同上

`tsconfig.json` に `"types": ["bun-types"]` を追加することで解決。

```json
{
  "compilerOptions": {
    "strict": true,
    "jsx": "react-jsx",
    "jsxImportSource": "hono/jsx",
    "types": ["bun-types"]
  }
}
```

`@types/bun` パッケージが Node.js 互換 API の型（crypto, Buffer 等）と `bun:test` モジュール型を提供しており、明示的に `bun-types` を指定することで `tsgo` が認識するようになる。

---

## 最終結果

```
82 pass
0 fail
148 expect() calls
Ran 82 tests across 6 files.
```

TypeScript エラー：0。全テスト通過。
