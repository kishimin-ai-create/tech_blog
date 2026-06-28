# `generateContentPreview` の改行コード正規化バグ修正 — `\r\n` / `\r` への対応

## エラー概要

`backend/src/models/diary.ts` にある `generateContentPreview` 関数は、日記本文を一行のプレビュー文字列に変換する純粋関数です。改行を空白に置き換えたあと 100 文字で切り詰め、必要に応じて `...` を付加します。

修正前の実装は **Unix LF (`\n`) にしか対応していませんでした**。

```ts
// 修正前
const processed = content.replace(/\n/g, " ");
```

Windows 環境で作成されたファイルや、HTTP リクエストで送られてくるテキストには **CRLF (`\r\n`)** が含まれることがあります。また、古い Mac 形式の **CR (`\r`) 単独** が混在するケースも存在します。これらは `/\n/g` では置換されないため、プレビュー文字列に `\r` がそのまま残るという問題が起きていました。

## 原因

正規表現 `/\n/g` は「LF 文字 1 つ」だけにマッチします。

| 改行コード | 意味 | 修正前の挙動 |
|-----------|------|------------|
| `\n` | Unix LF | ✅ 空白に置換される |
| `\r\n` | Windows CRLF | ❌ `\r` が残る |
| `\r` | 旧 Mac CR | ❌ 何も起きない |

`\r\n` の場合は `\n` だけが消え `\r` がそのまま文字列に残ります。端末やブラウザによっては表示が崩れたり、後続の文字列処理で予期しない動作を引き起こす原因になります。

## 修正内容

```ts
// 修正後（backend/src/models/diary.ts）
const processed = content.replace(/\r?\n|\r/g, " ");
```

正規表現 `/\r?\n|\r/g` は次の 3 パターンすべてにマッチします。

| パターン | 何にマッチするか |
|---------|----------------|
| `\r?\n`（`\r` あり） | Windows CRLF `\r\n` |
| `\r?\n`（`\r` なし） | Unix LF `\n` |
| `\r`（代替枝）| 旧 Mac CR `\r` |

`\r\n` は **1 つのマッチ** として処理されるため、`\r` と `\n` が個別に空白に置き換えられて二重スペースが生じる心配もありません。

## テストの追加

`backend/src/models/diary.small.test.ts` に `describe("Line Ending Normalisation", ...)` ブロックが追加されました。

```ts
describe("Line Ending Normalisation", () => {
  test("replaces Windows CRLF (\\r\\n) with a space", () => {
    expect(generateContentPreview("line one\r\nline two")).toBe("line one line two");
  });

  test("replaces old Mac CR (\\r) with a space", () => {
    expect(generateContentPreview("line one\rline two")).toBe("line one line two");
  });

  test("replaces multiple mixed line endings each with a space", () => {
    expect(generateContentPreview("a\nb\r\nc\rd")).toBe("a b c d");
  });
});
```

3 つのテストがカバーするケースは以下のとおりです。

1. **CRLF 単独** — Windows 形式が正しく 1 つの空白に変換される
2. **CR 単独** — 旧 Mac 形式が正しく 1 つの空白に変換される
3. **混在** — `\n` / `\r\n` / `\r` が混在していてもそれぞれ空白 1 つに変換される

3 つ目のテストは正規表現の境界条件を検証しており、`\r\n` が二重スペースになっていないことも間接的に確認しています。

修正後、テストスイート全体で **85 件合格・0 件失敗** となり、TypeScript のコンパイルエラーもありません（コミット `7eb6024`）。

## まとめ

- `generateContentPreview` の `/\n/g` は Unix LF のみ対応しており、`\r\n` や `\r` はそのまま残っていた
- 正規表現を `/\r?\n|\r/g` に変更することで 3 種類の改行コードをすべてカバーした
- `\r\n` を 1 つのマッチとして扱うため、二重スペースが発生しない
- 各改行形式に対応するテスト 3 件を追加し、リグレッションを防ぐ

改行コードの正規化は「まず Unix LF だけ対応する」ことが多いですが、フォームや外部入力を扱うバックエンドでは早い段階で 3 種類に対応しておくと、環境依存の不具合を未然に防げます。
