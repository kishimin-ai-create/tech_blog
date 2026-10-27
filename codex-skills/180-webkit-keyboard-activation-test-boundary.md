# WebKitのTab失敗を、リンクへの到達とEnterでの実行に分けて調べる

## 結論

キーボード操作のE2Eが失敗したときは、対象へフォーカスできなかったのか、実行後の画面が間違っていたのかを分けて確認する。profile-hubではWindows上のWebKitでTabによるリンク到達が成立せず、最終テストでは開始位置を明示してEnterの実行と遷移後のフォーカスを検証した。

これはTab順序の検証を完了したという意味ではない。ローカルの根本原因は未確定のまま、保証できる範囲を限定した記録である。確認日は2026年10月27日、Playwright 1.62.1を使用した。

## 最初の失敗には準備待ちも含まれていた

最初のスキップリンクテストでは、画面の準備を待たずにTabを押し、ChromiumとWebKitの両方で失敗した。ホームの表示を待つようにするとChromiumは通ったが、WebKitには失敗が残った。

同じ失敗メッセージでも、待機不足と環境によるキー操作の違いが混ざり得る。まずページの準備を揃えてから、残る失敗を調べた。

## ローカルで観測したこと

Windows上のWebKitで新しいページを開き、Tab、続けてTab、Alt+Tabを試した。しかし、いずれも観測した`activeElement`はBODYだった。

[PlaywrightのIssue #5609](https://github.com/microsoft/playwright/issues/5609)には、MacのWebKitでOption/Altを伴うTabを使う説明がある。ただし、その説明を根拠にWindowsの設定が原因と断定することはできない。今回の環境ではAlt+Tabで解決した事実もない。

## テストの開始条件と検証対象を分けた

最終のテストでは表示を待ってから、対象リンクを明示的にfocusし、Enterで実行した。要点を抜き出すと次の手順になる。

```typescript
await link.focus();
await page.keyboard.press("Enter");
await expect(page.getByRole("main")).toBeFocused();
```

`link`は各操作で対象にするリンクのlocatorである。これで確認するのは、フォーカス済みのリンクをキーボードで実行でき、移動先の本文にフォーカスすること。Tabでそこへ到達できるかは別の検証として残る。

WebKit自体を対象から外したり、テスト全体をskipしたりはしていない。最終のブラウザテスト全体はChromiumとWebKitで計26件成功した。

## 残る課題

Tab順序が要件に含まれる場合は、実際にTab移動が成立する環境条件を確認し、その経路を別途検証する必要がある。今回の修正はその代わりにはならない。

テストを通す変更をしたときほど、「開始条件として与えたこと」と「assertionで確認したこと」を記録したい。開始位置を明示したなら、その位置へ利用者が到達できることまで確認したとは報告しない。

## 根拠

- [対象コミットのnavigation.spec.ts](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/e2e/navigation.spec.ts)
- [PR #51](https://github.com/kishimin-ai-create/profile-hub/pull/51)
- [本文へのフォーカス実装の記事179](179-spa-navigation-main-focus.md)
