# VitestがPlaywright E2EテストをピックアップしてCIが落ちた原因と対処

## エラー概要

CI上でフロントエンドの Vitest 実行が次のエラーで失敗するようになった。

```
Error: Playwright Test did not expect test() to be called here.
```

エラーメッセージが示す通り、Playwright 用の `test()` 関数が Playwright ランナー以外の環境（Vitest）から呼び出されたことが原因だ。

---

## 対象読者

- Vitest と Playwright を同一フロントエンドプロジェクトに共存させているエンジニア
- テストファイルの命名規則を変更したあとに CI が壊れて困っている人

---

## 発生の経緯

このリポジトリのテスト規約（`.github/instructions/test.instructions.md`）では、**すべてのテストファイル名にサイズプレフィックス**を付けることが義務付けられている。

```
*.small.test.ts(x)
*.medium.test.ts(x)
*.large.test.ts(x)
```

この規約に従い、Playwright のデフォルトファイル `frontend/e2e/example.spec.ts` を `frontend/e2e/example.medium.test.ts` にリネームした（詳細は別記事参照）。

リネーム後、`vitest.config.ts` に設定された `include` パターンがこのファイルを**新たにピックアップするようになった**。

```ts
// frontend/vitest.config.ts（修正前）
test: {
  include: ["**/*.{small,medium,large}.test.{ts,tsx}"],
  // e2e/ を除外する設定がなかった
}
```

`**/*.medium.test.ts` というパターンは `e2e/example.medium.test.ts` にマッチする。Vitest はこのファイルを通常のユニットテストとして実行しようとするが、ファイル内部では `@playwright/test` の `test()` を使っているため、Playwright ランナー外では実行できない。

---

## 原因の整理

| 要素 | 状態 |
|------|------|
| `include` パターン | `**/*.{small,medium,large}.test.{ts,tsx}` — 広範にマッチ |
| E2E ファイルのパス | `e2e/example.medium.test.ts` — パターンにマッチしてしまう |
| `exclude` 設定 | **未設定だった** |
| `e2e/` ディレクトリの役割 | Playwright 専用。Vitest では実行不可 |

リネームそのものは問題なく、**`vitest.config.ts` に `e2e/` の除外設定がなかった**ことが根本原因だ。

---

## 対処

`vitest.config.ts` の `test` ブロックに `exclude` を追加した。

```ts
// frontend/vitest.config.ts（修正後）
export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    include: ["**/*.{small,medium,large}.test.{ts,tsx}"],
    exclude: ["e2e/**/*"],   // ← 追加
    passWithNoTests: true,
    environment: "jsdom",
    globals: true,
    setupFiles: ["./vitest.setup.ts"],
    // ...
  },
});
```

`exclude` は Vitest のデフォルト除外リスト（`node_modules` など）を**上書きするのではなく追加**する形で機能するため、既存の除外設定に影響しない。

---

## なぜ `|| true` や `continue-on-error` で黙らせなかったのか

「どうせ Playwright 側でも実行されるからエラーを無視すればいい」という解決策も考えられるが、それは採用しなかった。

理由は**本物の失敗を隠すリスク**があるからだ。`exclude` で明示的にスコープを限定する方が、Vitest と Playwright それぞれの担当範囲を正確に表現でき、将来 `e2e/` に別のファイルが増えたときも安全に動く。

---

## まとめ

| 項目 | 変更前 | 変更後 |
|------|--------|--------|
| `exclude` 設定 | なし | `["e2e/**/*"]` |
| E2E テストの Vitest 実行 | ❌ 誤ってピックアップ | ✅ 除外される |
| Playwright ランナー | 変更なし | 変更なし |

`include` パターンを広く設定しているプロジェクトでは、**Vitest に拾わせたくないディレクトリを `exclude` で明示**しておく必要がある。ツールの規約を満たすためにファイルをリネームするとき、他のツールのグロブパターンに意図せずマッチしないかを確認する習慣も重要だ。

---

## 関連記事

- `e2e-spec-ts-renamed-to-medium-test-ts.md` — `example.spec.ts` を `example.medium.test.ts` にリネームした経緯
