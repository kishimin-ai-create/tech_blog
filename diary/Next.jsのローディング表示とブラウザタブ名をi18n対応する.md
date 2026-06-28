# Next.jsのローディング表示とブラウザタブ名をi18n対応する

## 対象読者

- Next.js と React でアプリ内言語切替を実装している人
- 通常画面は翻訳できているのに、ローディングやブラウザタブ名だけ翻訳漏れする問題を避けたい人
- `next-intl` を使った UI の退行テストを増やしたい人

## この記事で扱うこと

この記事では、日記アプリのフロントエンドで発生した i18n 漏れを扱います。

対象は次の2点です。

- ローディング表示が現在の locale に追従しない
- ファビコン横に表示されるブラウザタブ名が日本語固定になる

認証、API 接続、DB、ルーティング設計の詳細は扱いません。

## 背景

このアプリでは、ヘッダーの言語切替で日本語と英語を切り替えられます。通常のナビゲーションやロゴの alt テキストは `next-intl` のメッセージを通して表示していました。

一方で、ローディング中の表示やブラウザタブ名は通常画面とは別の経路で表示されます。そのため、通常表示だけを確認していると翻訳漏れに気づきにくくなります。

実際に問題になっていたのは、次のような箇所でした。

- `frontend/app/page.tsx` の Suspense fallback が `messages.ja.diary.loading` を直接参照していた
- 詳細画面と管理編集画面のローディングが個別の `<p>` 表示になっていた
- `frontend/app/layout.tsx` の metadata は日本語固定で、アプリ内の言語切替後もブラウザタブ名が追従しなかった

## 原因

原因は、ローディングとブラウザタブ名が「通常の翻訳コンテキスト」を通らない実装になっていたことです。

通常のコンポーネントでは `useTranslations` を使えば現在の locale に応じた文言を取得できます。しかし、Suspense fallback で直接 `messages.ja` を参照すると、英語表示中でも日本語の固定文言が出ます。

また、ブラウザタブ名は Next.js の `metadata` で初期値を設定できますが、このアプリの locale はクライアント側の select で切り替えています。つまり、クライアント側の locale 変更に合わせて `document.title` も更新しないと、ファビコン横のサービス名だけ日本語のまま残ります。

## 対応

ローディング表示は、既存の共通コンポーネント `DiaryLoadingStatus` に寄せました。

このコンポーネントは `useTranslations("app")` と `useTranslations("diary")` を使うため、表示中の locale に合わせて次の内容が切り替わります。

- ローディング文言
- ロゴ画像の alt テキスト
- 全画面ローディングの `role="status"`

変更した主なファイルは次の通りです。

- `frontend/app/page.tsx`
- `frontend/app/diaries/[id]/page.tsx`
- `frontend/app/admin/edit/[id]/page.tsx`
- `frontend/app/features/diary/components.tsx`

ブラウザタブ名については、アプリの provider 側で locale 変更時に `document.title` を更新するようにしました。

```tsx
useEffect(() => {
  document.title = messages[locale].app.name;
}, [locale]);
```

これにより、日本語では `つづる日記`、英語では `Daybook` がブラウザタブに表示されます。

## テストで守ったこと

今回の修正では、通常の日本語表示だけでなく、英語 locale のローディングもテストしました。

追加した確認は次の通りです。

- ホーム画面の Suspense fallback が英語 locale で `Loading diaries.` を表示する
- 詳細画面のローディングが英語 locale で `Daybook logo` を表示する
- 管理編集画面のローディングも同じ共通コンポーネントを使う
- 言語切替後に `document.title` が `Daybook` へ変わる

実行した検証は次の通りです。

```bash
bun run typecheck
bun run lint
bun run test
```

最終的に、フロントエンドのテストは 68 件すべて通りました。

## 再発防止

同じミスを避けるため、FixFrontendAgent と関連 skill にもガードを追加しました。

追加した観点は次の通りです。

- ローディング表示を修正するときは、非デフォルト locale でも確認する
- ローディング文言だけでなく、ロゴ alt テキストも確認する
- ファビコン横のブラウザタブ名も visible UI として扱う
- render fallback で `messages.ja` のような default locale 直接参照を避ける

対象ファイルは次の通りです。

- `.agents/skills/fix-patterns/SKILL.md`
- `.agents/skills/frontend-components/SKILL.md`
- `.agents/skills/frontend-testing/SKILL.md`
- `.codex/agents/FixFrontendAgent.toml`
- `.github/agents/FixFrontendAgent.agent.md`
- `.github/skills/*`

## まとめ

i18n 対応では、通常画面の文言だけでなく、ローディング、画像 alt、ブラウザタブ名も確認対象にする必要があります。

特に Suspense fallback や metadata は、通常の画面コンポーネントと違う経路で表示されるため、default locale の直接参照が入り込みやすい場所です。

今回の修正では、ローディング表示を共通コンポーネントに寄せ、タブ名を locale と同期し、さらに Fix 系エージェントのチェックリストにも同じ観点を追加しました。次に同じ種類の修正をするときは、テストとエージェント指示の両方で翻訳漏れを拾える状態になります。
