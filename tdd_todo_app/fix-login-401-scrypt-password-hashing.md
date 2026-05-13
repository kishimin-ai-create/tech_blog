# ログインが常に 401 を返すバグを scrypt ハッシュ化で修正した話

## エラー概要

`POST /api/v1/auth/login` に正しいメールアドレスとパスワードを送っても、レスポンスが常に `401 UNAUTHORIZED`（`INVALID_CREDENTIALS`）になるというバグが発生していた。

```
HTTP/1.1 401 Unauthorized
{
  "code": "UNAUTHORIZED",
  "message": "Invalid email or password."
}
```

パスワードが正しいにもかかわらず認証が通らないため、ログイン機能が事実上使えない状態だった。

---

## 原因

問題の根源は `backend/src/services/auth-interactor.ts` の **`signup()` 関数**にあった。

```ts
// 修正前
const user = {
  id: generateId(),
  email: input.email,
  token: generateId(),
  passwordHash: input.password,   // ← ハッシュ化せず生のパスワードをそのまま保存
};
```

フィールド名は `passwordHash` だが、実際には**ハッシュ処理が行われておらず、平文のパスワードが保存されていた**。

`login()` 側も同様に素朴な実装だった。

```ts
// 修正前
if (!user) {
  throw new AppError('UNAUTHORIZED', 'Invalid email or password.');
}
if (user.passwordHash !== input.password) {   // ← 文字列の直接比較
  throw new AppError('UNAUTHORIZED', 'Invalid email or password.');
}
```

この構成は「signup も login も平文を扱う限りは動く」という一見無害な状態だった。  
しかし、**DB にすでに正しく scrypt ハッシュ化されたパスワードが保存されている場合**（例：テストで `buildScryptHash()` を使って事前投入した場合）、`"salt:hash_bytes" !== "rawPassword"` が必ず `true` になるため、**正しいパスワードを入力しても必ず 401 になる**というバグが顕在化した。

---

## 修正内容

### 1. `hashPassword()` の追加

Node.js 組み込みの `crypto` モジュールを使って scrypt ベースのハッシュ関数を実装した。**npm パッケージの追加は一切不要**。

```ts
import { randomBytes, scryptSync, timingSafeEqual } from 'node:crypto';

const SALT_LENGTH = 16;
const KEY_LENGTH = 64;

function hashPassword(password: string): string {
  const salt = randomBytes(SALT_LENGTH).toString('hex');   // 16 バイト → 32 文字の hex
  const key = scryptSync(password, salt, KEY_LENGTH).toString('hex');
  return `${salt}:${key}`;   // 保存フォーマット: "saltHex:keyHex"
}
```

- ソルトは毎回ランダム生成（レインボーテーブル攻撃対策）
- 導出鍵は 64 バイト（512 ビット）
- 保存文字列は `${saltHex}:${keyHex}` 形式で DB に保存

### 2. `verifyPassword()` の追加

```ts
function verifyPassword(password: string, storedHash: string): boolean {
  const separatorIndex = storedHash.indexOf(':');
  if (separatorIndex === -1) return false;

  const salt = storedHash.slice(0, separatorIndex);
  const key = storedHash.slice(separatorIndex + 1);

  const keyBuffer = Buffer.from(key, 'hex');
  const derivedKey = scryptSync(password, salt, KEY_LENGTH);

  return timingSafeEqual(keyBuffer, derivedKey);
}
```

`timingSafeEqual` を使うことで、比較にかかる時間がパスワードの内容に依存しない**定時間比較**を実現している。これにより、タイミングサイドチャネル攻撃を防ぐ。

### 3. `signup()` と `login()` の書き換え

```ts
// signup — 修正後
passwordHash: hashPassword(input.password),   // ハッシュ化してから保存

// login — 修正後
if (!user || !verifyPassword(input.password, user.passwordHash)) {
  throw new AppError('UNAUTHORIZED', 'Invalid email or password.');
}
```

`login()` では `!user` と `!verifyPassword()` を 1 つの条件にまとめることで、「ユーザーが存在しない」「パスワードが違う」のどちらの場合も同じエラーメッセージを返す（ユーザー存在有無の列挙攻撃対策）。

---

## TDD サイクル

このバグ修正は以下の TDD サイクルで進めた。

### 🔴 RED — バグを再現するテストを書く

`backend/src/tests/integrations/services/auth-interactor.medium.test.ts` に 4 つのテストを追加した。

| テスト | 確認内容 |
|---|---|
| `stores a scrypt hash, not the raw password` | signup 後、DB に生のパスワードではなく `:` を含むハッシュが保存されること |
| `returns user when raw password matches the stored scrypt hash` | scrypt ハッシュを事前投入した状態でログインが成功すること |
| `throws UNAUTHORIZED when password does not match` | 間違ったパスワードで 401 が返ること |
| `succeeds when user signs up then logs in` | signup → login のラウンドトリップが通ること |

修正前は「scrypt ハッシュ済みの認証情報でのログイン」と「signup 後に平文が保存されていないこと」の 2 テストが失敗することを確認した。

### 🟢 GREEN — 修正を適用してテストを通す

`hashPassword()` と `verifyPassword()` を実装し、`signup()` と `login()` に組み込んだ。  
修正後、**インテグレーションテスト 346 件・ユニットテスト 143 件すべてがパス**した。

---

## ポイントまとめ

| 観点 | 修正前 | 修正後 |
|---|---|---|
| パスワード保存 | 平文のまま | scrypt + ランダムソルトでハッシュ |
| ログイン比較 | `!==` 文字列比較 | `timingSafeEqual` による定時間比較 |
| 外部依存 | なし | なし（Node.js 組み込み `crypto` のみ） |
| タイミング攻撃耐性 | なし | あり |
| ユーザー列挙耐性 | 条件が分岐していた | 1 つの条件にまとめて同一エラー返却 |

---

## まとめ

フィールド名が `passwordHash` であっても、**実際にハッシュ処理を呼んでいなければ意味がない**。今回のバグは「命名と実装が乖離していた」ことで気づきにくい状態が続いていた典型例と言える。

修正には Node.js 組み込みの `crypto` モジュールだけを使い、外部ライブラリを追加せずにセキュリティ要件（ソルト付きハッシュ・定時間比較）を満たした。TDD によってバグを先にテストで明文化してから修正することで、修正の意図と効果が明確に検証できた。
