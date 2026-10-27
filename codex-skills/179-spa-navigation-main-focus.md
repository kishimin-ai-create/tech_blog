# SPAの画面遷移後に本文へフォーカスを移し、初回読み込みは維持する

## 結論

SPAでURLと画面だけを切り替えると、操作したリンク側にフォーカスが残り得る。profile-hubではpathnameが変わったときに`main`へフォーカスを移し、初回表示では移動しないようにした。

2026年10月27日に実装・確認した公開サイト基盤の記録である。React 19.2.8、TanStack Router 1.170.46を使用した。対象PR #51は未マージ。

## 初回表示と画面遷移を区別する

共通レイアウトは遷移前後で残るため、マウント時だけの処理では各ページへの移動に対応できない。一方、初回表示でも無条件にフォーカスすると、ブラウザが持つ初期状態を上書きする。

`SiteLayout`では前回のpathnameをrefに保持した。実装の要点は次の形である。

```tsx
const previousPath = useRef(pathname);
const mainRef = useRef<HTMLElement>(null);

useEffect(() => {
  if (previousPath.current !== pathname) {
    mainRef.current?.focus({ preventScroll: true });
    previousPath.current = pathname;
  }
}, [pathname]);
```

`main`には`ref`と`tabIndex={-1}`を設定した。通常のTab移動の停止先を増やさず、プログラムからはフォーカスできるようにしている。`preventScroll`を指定し、フォーカス操作によるスクロールを抑制する。

## スキップリンクも同じ本文へ向ける

共通headerの前に本文へ移動するリンクを置き、`href="#main-content"`で同じ`main`を指す。ルート変更時の処理と、利用者が選ぶスキップリンクは、どちらも本文へ到達するための経路になる。

今回の判定はpathnameの変更である。検索条件やhashだけが変わる操作には同じ処理を実行しない。将来、同じpathname内で主要な内容が大きく変わる画面を追加するときは、別の要件として扱う必要がある。

## ブラウザで確認したこと

公開ナビゲーションのE2Eでは、セクションの入口やContactをEnterで実行した後、`main`がフォーカスされることを確認した。スキップリンクの実行と、履歴で戻る操作も含めて確認した。最終のE2E全体はChromiumとWebKitで計26件成功した。

ただし、WebKitのテストでは開始リンクへの明示的なfocusを使っている。これはキーボード実行と遷移後のフォーカスの検証であり、Tabキーでの到達順序の証明とは分ける。この切り分けは[別の記事180](180-webkit-keyboard-activation-test-boundary.md)に記録した。

## 残る確認

フォーカスの位置をテストできても、すべての支援技術で読み上げが適切になることまでは確認していない。スクリーンリーダーの通し確認は未実施である。画面構造やルーティングを変更した場合も、この操作契約を確認し直す。

## 根拠

- [PR #51](https://github.com/kishimin-ai-create/profile-hub/pull/51)
- [対象コミットのSiteLayout](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/src/app/router/SiteLayout.tsx)
- [対象コミットの操作テスト](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/e2e/navigation.spec.ts)
