# React Routerの404とroot ErrorBoundaryを別の復旧境界として接続する

## 対象読者

React RouterとReactのError Boundaryを同じアプリケーションへ組み込む際に、404画面と予期しないエラー画面の責務を分けたい開発者を対象にする。

## 結論

存在しないURLはRouterの`notFoundComponent`から`NotFoundView`へ渡し、描画中の予期しない例外はアプリケーション最上位の`ErrorBoundary`から`ErrorFallback`へ渡す。両者を別の境界として接続すると、404ではホームへのリンクを提供し、予期しない例外ではページ再読み込みを提供できる。

## 404の経路

ルートツリーのnot-found設定は、未知のパスを`NotFoundView`へ渡す。`NotFoundView`は`I18nProvider`の文脈から表示言語を取得し、`Link`で`/`へのリンクを描画する。これは移動操作なので、見た目をボタン風にしてもHTML上の役割はリンクのままにする。

```tsx
<Link to="/">{messages.homeLink}</Link>
```

Smallテストでは、日本語・英語の見出しと説明、リンクのアクセシブルな名前、`href="/"`を確認する。Router内部の実装ではなく、利用者が見える文言と移動先をアサートする。

## 予期しない例外の経路

`AppProviders`の最外側に`ErrorBoundary`を置き、その内側に`QueryClientProvider`と`I18nProvider`を配置する。この順序により、Providerの子孫で発生した描画例外を`ErrorFallback`へ渡せる。`ErrorFallback`は通常のi18n Contextを使わず、Providerの外側でも読み込める最小辞書とロケール解決処理を利用する。

```tsx
<ErrorBoundary>
  <QueryClientProvider client={queryClient}>
    <I18nProvider>{children}</I18nProvider>
  </QueryClientProvider>
</ErrorBoundary>
```

ErrorFallbackはHeader/Footerを描画せず、「エラーが発生しました」と説明文、ページ再読み込みボタンを表示する。再読み込みはクライアントサイドのRouter遷移ではなく、`window.location.reload()`で初期状態から復旧する操作として扱う。

## 検証結果

対象ブランチでは、root routeのホーム表示、未知パスからのNotFoundView表示、ホームリンクのRouter遷移、AppProvidersの子要素例外からErrorFallbackへの切り替えを段階的なテストと実装で追加した。関連コミットは`e917b37`、`c9e6079`、`942198a`、`bd5e143`、`93f2bf8`である。

## 事実・判断・未確認事項

- FACT: `AppProviders`は`ErrorBoundary`を最外側に配置している。
- FACT: NotFoundViewは`Link`で`/`を参照する。
- FACT: ErrorFallbackはHeader/Footerを描画しない。
- INFERENCE: ルーティングエラーと描画例外を別の復旧操作へ分離できる。
- ASSUMPTION: 実ブラウザでの再読み込み後の初期表示は、Smallテストの対象外として別途確認が必要である。

## まとめ

404はRouterのナビゲーション境界、予期しない例外はReactの描画境界として扱う。テストでは内部の配線そのものではなく、各境界から利用者が観測できる画面と復旧操作を検証することで、責務の違いを明確に保てる。

## 参考

- `docs/v1/ui/ui.md §19–20`
- `docs/v1/ui/components/App.md`
- `docs/v1/ui/components/NotFoundView.md`
- `docs/v1/ui/components/ErrorFallback.md`
