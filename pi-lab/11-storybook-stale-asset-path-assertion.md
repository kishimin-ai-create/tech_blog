# StorybookのInteraction Testが古いアセットパスを検証し続けていた話

## 結論

アセットのファイル形式やパスを変更したら、そのアセットを検証しているテストのアサーションも同じコミットで更新する。pi-labでは、ヘッダーロゴをPNGからSVGへ切り替えたコミットがStorybookのInteraction Testの期待値を更新し忘れ、翌日のCIで無関係に見えるテスト失敗として表面化した。

## 発生した failure

CI（`npx vitest run`、GitHub Actions上）で次の1件が失敗した。

```text
FAIL   storybook (chromium)  src/components/header.stories.tsx > Default
Error:
expect(element).toHaveAttribute("src", "/src/assets/pi-lab-logo.png")
Expected the element to have attribute:
  src="/src/assets/pi-lab-logo.png"
Received:
  src="/src/assets/pi-lab-logo.svg"
```

`Received`に実際の値が出ている時点で、アプリのバグではなくテスト側の期待値が古い可能性を疑うべきシグナルだった。

## 原因調査

`src/components/header.tsx`を確認すると、ロゴの読み込みは次のようになっていた。

```tsx
import logoImage from "../assets/pi-lab-logo.svg";
```

`git log --oneline -- src/components/header.tsx`で履歴を辿ると、1つ前のコミットで変更されていた。

```text
commit db2dabb
    fix: address PR review comments
    ...
    - Switch header logo from PNG (2.1MB) to SVG (268KB)
```

```diff
- import logoImage from "../assets/pi-lab-logo.png";
+ import logoImage from "../assets/pi-lab-logo.svg";
```

一方、`src/components/header.stories.tsx`のInteraction Testは同じコミットで更新されておらず、`.png`を期待したままだった。

```tsx
await expect(image).toHaveAttribute("src", "/src/assets/pi-lab-logo.png");
```

つまりアプリのコードは意図通りSVGに切り替わっており、実際にリグレッションが起きていたのはテストの期待値の方だった。

## 修正

期待値をSVGへ合わせた。

```diff
- await expect(image).toHaveAttribute("src", "/src/assets/pi-lab-logo.png");
+ await expect(image).toHaveAttribute("src", "/src/assets/pi-lab-logo.svg");
```

`grep -rn "pi-lab-logo.png" src/`で他に参照がないことを確認したうえで、使われなくなった2.1MBの`src/assets/pi-lab-logo.png`も削除した。この削除自体はテスト修正の副次効果であり、修正の主目的ではない。

## 検証結果

```text
$ npx vitest run src/components/header.stories.tsx
 Test Files  1 passed (1)
      Tests  1 passed (1)

$ npx vitest run
 Test Files  12 passed (12)
      Tests  26 passed (26)
```

`npx eslint`と`npx tsc --noEmit`もエラーなしだった。修正コミットは`896d181`である。

## 学び

- テストが失敗したとき、`Expected`と`Received`のどちらが「あるべき状態」かを先に判断する。今回は`Received`（実際のDOM）の方が正しく、`Expected`（テストの期待値）が古かった。
- アセットのファイル名や形式を変えるコミットは、そのアセットに依存するテスト・Story・スナップショットを同じコミットでgrepし、漏れなく更新する。
- 未参照になったアセットファイルは、削除前に`grep`でリポジトリ全体を確認してから削除する。

## 参考資料

- 根拠コミット: `db2dabb`（ロゴ切り替え）、`896d181`（本修正）
- 確認日: 2026-08-07
