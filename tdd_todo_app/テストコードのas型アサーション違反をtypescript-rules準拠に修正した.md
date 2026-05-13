# テストコードの `as` 型アサーション違反を typescript.rules.md 準拠に修正した

## 対象読者

- TypeScript でテストコードを書いているエンジニア
- `as` アサーションの「必要な使用」と「不要な使用」の判断に迷っている人
- プロジェクト内のリントルールや規約をテストコードにも適用したい人

---

## 背景

このリポジトリには `typescript.rules.md` というルールファイルがあり、次のルールを定めている。

> `as` による型アサーションは**禁止**。  
> どうしても使う必要がある場合は、**直前の行に理由を説明するコメントを必ず付ける**。

コードレビュー対応（コミット `4ac3070`）で統合テストを整備した際、  
`as Record<string, unknown>` が3つのテストファイルに紛れ込んでいた。  
また本番コードの `request-validation.ts` にも `as` アサーションがあったが、  
こちらは理由コメントが抜けていた。

コミット `50d59cb` でこれらを一括修正した。

---

## 問題：不要な `as` キャストがテストコードに混入していた

### 違反パターン

```typescript
// http-presenter.test.ts（修正前）
const dto = presentApp(app) as Record<string, unknown>;
expect(dto).not.toHaveProperty('deletedAt');
```

```typescript
// app-controller.test.ts（修正前）
expect((res.body.data as Record<string, unknown>)).not.toHaveProperty('deletedAt');
```

```typescript
// todo-controller.test.ts（修正前）
expect((res.body.data as Record<string, unknown>)).not.toHaveProperty('deletedAt');
```

いずれも `toHaveProperty()` を呼ぶ前に `as Record<string, unknown>` でキャストしている。

### なぜ不要か

Vitest の `toHaveProperty()` は型に関係なく動作する。  
引数は `unknown` 型を受け付けており、`as` でキャストしなくてもアサーションは正常に動く。

```typescript
// expect の型シグネチャ（概略）
expect(value: unknown): JestExpect
```

`presentApp(app)` の戻り値はすでに型付きのオブジェクトであり、  
`as Record<string, unknown>` にキャストしても **型安全性は何も向上しない**。  
むしろ TypeScript の型チェックを迂回して型情報を捨てているだけだった。

---

## 修正：不要なキャストを削除する

```typescript
// http-presenter.test.ts（修正後）
const dto = presentApp(app);
expect(dto).not.toHaveProperty('deletedAt');
```

```typescript
// app-controller.test.ts（修正後）
expect(res.body.data).not.toHaveProperty('deletedAt');
```

```typescript
// todo-controller.test.ts（修正後）
expect(res.body.data).not.toHaveProperty('deletedAt');
```

キャストを取り除いても、テストの意図・動作・結果は一切変わらない。  
それでいて型情報が保たれ、ルールに準拠した状態になる。

---

## 本番コード側：`as` が必要だったケース

本番コードの `request-validation.ts` には `as` が残っている。  
こちらは削除するのではなく、**理由コメントを追加**して対応した。

```typescript
// request-validation.ts（修正後）
function toRecord(value: unknown): Record<string, unknown> {
  if (typeof value !== 'object' || value === null || Array.isArray(value)) {
    return {};
  }

  // value has been narrowed to a non-null, non-array object by the guards above; cast to Record is safe
  return value as Record<string, unknown>;
}
```

`typeof value !== 'object'`、`value === null`、`Array.isArray(value)` の3つのガードを通過した時点で、  
`value` は「null でも配列でもないオブジェクト」に絞り込まれている。  
TypeScript は `object` 型をそれ以上 `Record<string, unknown>` に自動ナローイングしないため、  
`as` キャストが必要となる正当な場面だ。

ルールはこのケースを禁止していない。「**必要なら使ってよい、ただし理由を書け**」という運用になっている。

---

## `as` アサーションの使用判断フロー

```
as を使おうとしている
  │
  ├─ as なしで型チェックを通せるか？
  │     └─ YES → as は不要。削除する
  │
  └─ NO（型ガードや型推論で対応できない）
        └─ 直前の行に理由コメントを書いてから as を使う
```

ポイントは「型安全性に貢献しているか」だ。  
テストコードでの `as Record<string, unknown>` は Vitest の `expect` を通すためのものではなく、  
書いた人の習慣や勘違いによって入り込んだケースが多い。

---

## まとめ

| ファイル | 対応内容 | 理由 |
|---|---|---|
| `http-presenter.test.ts` | `as Record<string,unknown>` を削除 | `toHaveProperty` はキャスト不要 |
| `app-controller.test.ts` | `as Record<string,unknown>` を削除 | 同上 |
| `todo-controller.test.ts` | `as Record<string,unknown>` を削除 | 同上 |
| `request-validation.ts` | 理由コメントを追加 | ガードで絞り込み済み・`as` が正当 |

`typescript.rules.md` の規約はコードの品質を上げるためのものだが、  
テストコードには見落とされがちだ。  
規約をプロダクションコードと同じ水準でテストコードにも適用することで、  
「なぜこの `as` があるのか」という疑問を生まない、読みやすいテストスイートを維持できる。
