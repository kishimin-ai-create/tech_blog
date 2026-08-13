# ESLint flat config でスプレッド展開したプラグインが上書きされ Testing Library のルールが無効化されていたバグを修正した

## はじめに

結果として、`plugins` キーには `vitest` のみが残り、`rules` キーには Vitest のルールのみが残った。**Testing Library のプラグインもルールも、完全に無効化された状態で ESLint が実行されていた。**

## 対象読者

- ESLint の flat config（`eslint.config.mjs`）を使っているプロジェクトで複数のプラグインをテストファイルに適用したい人
- `eslint-plugin-testing-library` と `@vitest/eslint-plugin` を共存させようとしている人
- 「設定を書いたはずなのにルールが効いていない」という現象に遭遇したことがある人

---

## 背景

フロントエンドのテストファイルには、以下の 2 つのプラグインを ESLint で適用する意図があった。

| プラグイン | 目的 |
|---|---|
| `eslint-plugin-testing-library` | `getByRole` など Testing Library の正しい使い方を強制する |
| `@vitest/eslint-plugin` | `vitest/no-focused-tests` など Vitest 固有のルールを強制する |

設定は `eslint.config.mjs` のテストファイル向けオブジェクトに以下のように書かれていた。

```js
// ❌ 修正前
{
  files: ["**/*.test.{ts,tsx}", ...],
  ...testingLibrary.configs["flat/react"],   // ← オブジェクトレベルでスプレッド
  plugins: {
    vitest,                                   // ← 明示的な plugins キー
  },
  rules: {
    ...vitest.configs.recommended.rules,
    "vitest/max-nested-describe": ["error", { max: 3 }],
    // ...
  },
},
```

`bun run lint` はエラーなく通っており、一見問題のない設定に見えた。

---

## 根本原因：JavaScript オブジェクトのキー上書き

ESLint flat config の 1 エントリは **通常の JavaScript オブジェクト** である。このオブジェクトに対してスプレッド展開と明示的なキー定義を混在させると、同名キーは後勝ちで上書きされる。

`testingLibrary.configs["flat/react"]` は内部的に次のような構造を持つ。

```js
{
  plugins: { "testing-library": testingLibrary },
  rules: {
    "testing-library/await-async-queries": "error",
    // ... 多数のルール
  },
}
```

これをオブジェクトレベルでスプレッドすると `plugins` と `rules` が一度展開される。しかしその**直後**に明示的な `plugins: { vitest }` と `rules: { ... }` を書くと、JavaScript オブジェクトリテラルの仕様どおり後から書かれたキーが前のキーを**完全に置き換える**。

```js
// JavaScript オブジェクトの基本動作
const obj = {
  ...{ a: 1, b: 2 },  // a=1, b=2 が展開される
  b: 99,              // b が 99 に上書きされる → a=1, b=99
};
```

結果として、`plugins` キーには `vitest` のみが残り、`rules` キーには Vitest のルールのみが残った。**Testing Library のプラグインもルールも、完全に無効化された状態で ESLint が実行されていた。**

### なぜ lint エラーにならなかったのか

Testing Library のルールはコードの誤りを検出するルールであり、ルール自体が登録されていなければ何も報告されない。設定の書き方が間違っていても ESLint は警告を出さないため、ルールが無効化されていることは lint 実行結果だけでは気づきにくい。

---

## 実際にやったこと

`plugins` と `rules` それぞれのキーの**値の中で**両方をスプレッドするように変更した。

```js
// ✅ 修正後
{
  files: ["**/*.test.{ts,tsx}", ...],
  ...testingLibrary.configs["flat/react"],     // files 以外の共通設定は維持
  plugins: {
    ...testingLibrary.configs["flat/react"].plugins,  // ← Testing Library を明示展開
    vitest,
  },
  rules: {
    ...testingLibrary.configs["flat/react"].rules,    // ← Testing Library を明示展開
    ...vitest.configs.recommended.rules,
    "@typescript-eslint/no-unsafe-call": "off",
    "@typescript-eslint/no-unsafe-member-access": "off",
    "vitest/max-nested-describe": ["error", { max: 3 }],
    "vitest/no-focused-tests": "error",
    "vitest/no-disabled-tests": "warn",
  },
  settings: {
    vitest: { typecheck: true },
  },
  languageOptions: { globals: { ...vitest.environments.env.globals } },
},
```

ポイントは以下の 2 点。

1. **オブジェクトレベルのスプレッド `...testingLibrary.configs["flat/react"]`** はそのまま残す。これにより `name` など `plugins`・`rules` 以外のフィールドが引き継がれる。
2. **`plugins` キーと `rules` キーの値の中**でそれぞれ Testing Library の分を先にスプレッドし、Vitest の分を後から加える。これにより両者が共存する。

---

## 注意点：スプレッドの「どこで」が重要

ESLint flat config を扱う際にありがちな混乱を整理する。

| 書き方 | 動作 |
|---|---|
| `...plugin.configs.xxx` をオブジェクトレベルで展開し、その後同じキーを書く | **後から書いたキーで上書きされる（今回のバグ）** |
| `plugins: { ...plugin.configs.xxx.plugins, anotherPlugin }` とキー内で展開 | **両方が plugins に登録される（正しい）** |
| `extends: [plugin.configs.xxx]` に配列で渡す | **flat config の `extends` として別オブジェクト扱いになり安全** |

`extends` 配列を使う方法は ESLint が内部でマージを行うため、プラグインのスプレッドを自分で書く必要がない。今回はテストファイルだけに絞った細かい制御が必要だったため直接オブジェクトを書いていたが、将来的に複数プラグインを混在させる際は `extends` 配列の利用も検討に値する。

---

## 検証

修正後、`frontend/` ディレクトリで `bun run lint` を実行し、exit code 0 で完了することを確認した。

---

## まとめ

- ESLint flat config のオブジェクトは **通常の JavaScript オブジェクト** であり、同名キーは後勝ちで上書きされる
- プラグインの `configs["flat/xxx"]` をオブジェクトレベルでスプレッドした後に同名キーを追加すると、**プラグインとルールが丸ごと無効化される**
- 複数プラグインを共存させるには、**`plugins` と `rules` の各キーの値の中でスプレッドする**のが正しいパターン
- lint が通っているからといってすべてのルールが正しく登録されているとは限らない。設定の意図を確認する際は、登録されているプラグインとルールを `--print-config` オプションや `eslint --inspect-config` で確認するのが確実
