# UTCの日時をブラウザのタイムゾーンでずらさず表示する

## 対象読者

- API が UTC の ISO 8601 文字列を返しているアプリを作っている人
- DB の `created_at` / `updated_at` と画面表示がずれる問題を調べている人
- `Intl.DateTimeFormat` の timezone 指定を忘れがちな人

## 問題

DB の作成日時・更新日時と、日記一覧に表示される作成日時・更新日時がずれていた。

API は `createdAt` / `updatedAt` を UTC の ISO 8601 文字列として返している。一方、フロントエンドの表示は `Intl.DateTimeFormat("ja-JP")` のデフォルトタイムゾーンに依存していた。

日本時間の環境では、UTC の `09:00` が `18:00` として表示される。

## 原因

`frontend/app/features/diary/components.tsx` の `formatDateTime` は、`new Date(value)` を `Intl.DateTimeFormat` でフォーマットしていた。

`timeZone` を指定しない場合、表示は実行環境のローカルタイムゾーンになる。DB/API の UTC 値と比較したい画面では、この暗黙変換がズレとして見える。

## 対応

`formatDateTime` に `timeZone: "UTC"` を指定した。

```ts
new Intl.DateTimeFormat("ja-JP", {
  dateStyle: "medium",
  timeZone: "UTC",
  timeStyle: "short",
})
```

これにより、API から受け取った UTC 時刻を、ブラウザのローカルタイムゾーンへ変換せずに表示する。

## テスト

`frontend/app/features/diary/components.small.test.tsx` に、UTC の `createdAt` / `updatedAt` がそのまま一覧に表示されることを確認するテストを追加した。

修正前は `2026-06-22T09:00:00.000Z` が `18:00` と表示され、テストが失敗した。修正後は `9:00` として表示される。

## まとめ

日時表示は、API の値、DB の型、ブラウザのロケール、タイムゾーン指定が少しずつ影響する。DB/API の UTC 値と画面表示を一致させたい場合は、`Intl.DateTimeFormat` に `timeZone` を明示するのが重要になる。

