# Zod の `min(1)` でログインスキーマの空パスワードをバリデーション層で弾く

## はじめに

loginSchema.safeParse() → 成功（min がないので空文字列を通過）

## 対象読者

- Hono + Zod でバックエンド API を実装しているエンジニア
- 認証エンドポイントの入力バリデーション設計を考えているエンジニア
- scrypt など CPU コストの高い処理を含むサービス層を持つ開発者

---

## 背景

`backend/src/controllers/auth.controller.ts` のログインスキーマは次のように定義されていた。

```ts
const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().max(255), // min がない
});
```

`z.string()` はデフォルトで長さ 0 の文字列を許容する。
そのため `password: ""` というリクエストがバリデーションを通過し、
`AuthService.login()` まで流れ込んでいた。

---

## 原因：空パスワードが scrypt 演算を無駄に起動していた

空パスワードが `loginSchema.safeParse()` を通過すると、次のフローが実行される。

```
リクエスト: { email: "...", password: "" }
     ↓
loginSchema.safeParse() → 成功（min がないので空文字列を通過）
     ↓
AuthService.login() 呼び出し
     ↓
DB からユーザーをメールアドレスで検索
     ↓
verifyPassword("", stored) 呼び出し
     ↓
scryptSync("", salt, 64) 実行  ← CPU 負荷の高い暗号演算
     ↓
ハッシュ不一致 → 401 を返す
```

空文字列であることは自明に検出できるにもかかわらず、コストの高い scrypt 演算を
毎回実行してから 401 を返していた。

---

## 解決策：`min(1)` をバリデーション層に追加する

```diff
 const loginSchema = z.object({
   email: z.string().email(),
-  password: z.string().max(255),
+  password: z.string().min(1).max(255),
 });
```

`min(1)` を追加することで、空文字列のパスワードはバリデーション層で即座に弾かれる。

| | 修正前 | 修正後 |
|---|---|---|
| 空パスワードのレスポンス | 401（scrypt 演算後） | 400（バリデーション層で即時返却） |
| scrypt 演算の実行 | される | されない |

---

## 実装詳細：`registerSchema` との比較

同ファイルの `registerSchema` はすでに `min(8)` を持っていた。

```ts
const registerSchema = z.object({
  name: z.string().trim().min(1).max(50),
  email: z.string().email().max(255),
  password: z
    .string()
    .min(8)     // 登録時は最低 8 文字
    .max(255)
    .refine((p) => /[a-zA-Z]/.test(p), "must include a letter")
    .refine((p) => /[0-9]/.test(p), "must include a number"),
});
```

`loginSchema` には `min` が抜けており、定義の一貫性も欠けていた。

### なぜ `min(8)` ではなく `min(1)` か

ログインは「ユーザーが入力したパスワードを照合する」処理であり、
パスワードポリシーを強制する場所ではない。

仮に `loginSchema` に `min(8)` を置いてしまうと、過去に何らかの理由で
8 文字未満のパスワードで登録されたアカウントがログインできなくなるリスクがある。
バリデーションの目的は「空文字列かどうか」の検出に留めるべきであり、
`min(1)` はその最小限の要求を満たす修正となる。

---

## 注意点

**HTTP ステータスコードの変化に注意**

修正前は空パスワードに対して 401 を返していたが、修正後は 400 を返す。

- 401: 認証に失敗（有効な入力だったが認証情報が一致しなかった）
- 400: 不正な入力（そもそも有効なリクエストでない）

空パスワードは「入力が不正」であり 400 が意味的に正しい。
クライアント側のテストやエラーハンドリングがステータスコードに依存している場合は、
この変化を確認しておく必要がある。

**空パスワードとユーザー列挙への影響**

このプロジェクトの `AuthService` は、メールアドレスが存在しない場合とパスワードが
間違っている場合で同一のエラーメッセージ `"Invalid email or password."` を返すことで
ユーザー列挙攻撃（user enumeration）を防いでいる。

空パスワードは 400 としてより早い段階で弾かれるため、この設計に影響しない。
バリデーションエラーとしての 400 は、メールアドレスの存在有無に関係なく返るためだ。

---

## まとめ

| 項目 | 内容 |
|---|---|
| 変更ファイル | `backend/src/controllers/auth.controller.ts` |
| 変更内容 | `loginSchema` の `password` に `min(1)` を追加 |
| 修正前の挙動 | 空パスワードが 401 を返す（scrypt 演算が先に走る） |
| 修正後の挙動 | 空パスワードが 400 をバリデーション層で即時返却 |
| テスト結果 | 82/82 パス |
| TypeScript | エラー 0 件 |

バリデーション層は「無駄なサービス処理を防ぐゲートキーパー」でもある。
入力が明らかに不正なケースは、できるだけ早い段階で弾く設計が堅牢性とパフォーマンスの両面で好ましい。
