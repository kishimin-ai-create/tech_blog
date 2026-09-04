# モバイルの横スクロールを`min-width`から切り分ける

## 対象読者

レスポンシブ対応したWeb画面で、狭いviewportに横スクロールバーが表示される原因を調査する開発者。

## スコープ

Mojicaで発生した320px未満のviewportにおける横スクロールを扱う。すべてのoverflow問題に対する一般解や、ブラウザごとの描画差は扱わない。

## 症状と調査

スマートフォン表示で横スクロールバーが残った。フォームの縮小を妨げる要素として、CSSの`body`に`min-width: 20rem`が設定されていた。20remは320pxなので、viewportが320px未満でも文書幅が320pxを下回れない。

## 修正

`body`の最小幅指定を削除した。

```css
body {
  margin: 0;
}
```

画面側の`min-w-0`と`overflow-x-clip`は維持している。前者はflexアイテムが親幅より小さくなれるようにし、後者は画面外へ出た装飾などを横スクロール領域にしないための境界である。ただし、これらを追加しても`body`自身の最小幅制約は解除できないため、最終的には根元の`min-width`を確認する必要がある。

## 検証

| コマンド | 結果 |
| --- | --- |
| `bun run test` | 21ファイル、103テスト成功 |
| `bun run typecheck` | 成功 |
| `bun run lint` | 成功 |

狭いviewportでの実ブラウザ計測は今回のコマンドには含まれていないため、横スクロールがすべてのブラウザで解消したことは未確認である。

## 事実・判断・未確認事項

- FACT: `frontend/src/styles/globals.css`から`body { min-width: 20rem; }`を削除した。
- FACT: 修正前は320px未満でも文書の最小幅が320pxだった。
- INFERENCE: 横スクロールの一次原因は、viewportより大きい文書最小幅だったと判断できる。
- 未確認事項: Playwrightなどによる複数端末幅の実ブラウザ検証は未実施である。

## まとめ

横スクロールを見つけたら、まず子要素だけでなく`html`や`body`を含む文書幅の制約を確認する。`overflow`で隠す前に、viewportより大きな`min-width`が不要に残っていないかを取り除くことが、原因に近い修正になる。

## 参考

- `frontend/src/styles/globals.css`
- `92eb0de fix: prevent mobile form overflow`
- `4957c11 fix: clip mobile horizontal overflow`
- `afe0b3b fix: remove the mobile minimum width`
