# React 19のメタデータを使い、末尾スラッシュまでページ情報を揃える

## はじめに

React 19では、コンポーネントからtitleやmetaを描画し、文書のheadへ配置できます。profile-hubではこれを使い、公開ルートごとのタイトル・説明・Open Graph情報を設定しました。[Reactのmetaリファレンス](https://react.dev/reference/react-dom/components/meta)に記載されている動作です。

確認日は2026年10月27日です。対象はReact 19.2.8、TanStack Router 1.170.46のSPAで、[PR #45](https://github.com/kishimin-ai-create/profile-hub/pull/45)として公開しています。未マージの実装を題材にしています。

## やってみた結果

5ルートと未知のパスでタイトルを切り替え、クライアント側のリンク遷移でも説明やOGP情報を更新できました。画像を持つメタデータから画像を持たないものへ再描画したとき、以前のog:imageが残らないこともテストしています。

## 実装・検証

PageMetadataはtitle、description、任意のimageを受け取ります。次は実装の要点を抜き出したものです。

```tsx
<title>{title + " | profile-hub"}</title>
<meta name={"description"} content={description} />
<meta property={"og:title"} content={title + " | profile-hub"} />
{image && <meta property={"og:image"} content={image} />}
```

descriptionやog:descriptionも同じコンポーネントで描画します。index.htmlの固定titleは削除し、表示するタイトルの責務を揃えました。[titleの公式資料](https://react.dev/reference/react-dom/components/title)でも、同時に複数のtitleを描画しないよう説明されています。

ルートごとの情報はページ用の対応表へ置き、SiteLayoutが現在のpathnameから選びます。

## つまずいたところ

最初はpathnameをそのままキーにしていました。すると、ブラウザでHobbyの画面を表示できる`/hobby/`に直接アクセスしても、対応表に一致せず404用タイトルになりました。

URLの末尾スラッシュを取り除き、空文字列になった場合だけホームのキーへ戻す処理を追加しました。

```ts
pages[pathname.replace(/\/+$/, "") || "/"];
```

`/`を消したままにしないことも必要です。対応表に存在しないパスには、引き続き404のメタデータを返します。

## 動作確認

ブラウザテストでは直接アクセス、`/hobby/`、未知のURL、ホームからHobbyへのクライアント遷移を確認しました。タイトルの切り替えに加えて、og:descriptionが重複しないことも検証しています。メタデータ変更を含む全E2Eは、最終的にChromium・WebKitで26件成功しました。

## 学んだことと制約

画面が同じルートとして扱うURLを、メタデータの対応表だけが別物として扱うと、画面と文書情報が食い違います。表示ルートの確認にタイトルも含めることで、この不整合を発見できました。

この実装はクライアント描画の基盤です。クローラーが受け取る初期HTMLや、SNSのプレビュー表示を検証したものではありません。prerenderingなどのSEO対応は別Issueに残っています。

## 参考資料

- [PageMetadataとそのテスト](https://github.com/kishimin-ai-create/profile-hub/tree/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/src/components)
- [メタデータのブラウザテスト](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/e2e/metadata.spec.ts)

確認日: 2026年10月27日
