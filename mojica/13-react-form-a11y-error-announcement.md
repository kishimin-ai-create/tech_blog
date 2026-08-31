# Reactフォームの入力エラーを支援技術へ伝える設計

## 要約

Reactのフォーム部品で、入力エラーを見た目だけでなく支援技術にも伝えるための実装を整理した。`aria-errormessage`でエラー要素を参照し、動的に表示されるメッセージを`aria-live="polite"`で通知する。あわせて、エラー背景と説明文のコントラストを確認し、WCAG AAの基準を下回る色指定をforegroundトークンへ変更した。

## 背景

Mojicaの共有UIでは、TextFieldやColorPickerFieldが入力エラーを表示する。エラー文が画面に表示されるだけでは、スクリーンリーダー利用者が更新を知るタイミングや、どの入力に対応するエラーかを判断しにくい。また、AlertBannerの破壊的状態では、赤系の説明文と背景の組み合わせが通常サイズ文字のコントラスト基準を満たしていなかった。

## 採用した実装

### フィールドエラー

入力要素からエラー要素を参照する属性を、通常の補足説明とは分離した。

```tsx
<input
  aria-invalid={Boolean(errorMessage)}
  aria-errormessage={errorMessage ? `${inputId}-error` : undefined}
/>
<p id={`${inputId}-error`} aria-live="polite">
  {errorMessage}
</p>
```

`aria-errormessage`は値が無効なときに使い、参照先のエラー要素は利用者が確認できる状態にする。補足説明は`aria-describedby`で別に関連付ける。これにより、入力形式の説明と、現在の検証エラーを同じ説明文へ混在させずに済む。

### AlertBannerの説明文

Alertのvariantが設定する赤系の説明文色を、コンポーネント側で`text-foreground`へ上書きした。

```tsx
<AlertDescription className={"!text-foreground"}>
  {description}
</AlertDescription>
```

これは黒色を固定する変更ではなく、テーマのforegroundトークンを使う変更である。MDNが紹介するWCAG 1.4.3では、通常サイズの文字と背景に4.5:1以上のコントラストが求められるため、テーマごとの実測結果に基づいて色を選ぶ必要がある。

## 検証

実装後、次の検証を行った。

| コマンド                   | 結果                                                                     |
| -------------------------- | ------------------------------------------------------------------------ |
| `bun run typecheck`        | 成功                                                                     |
| `bun run lint`             | 成功、警告なし                                                           |
| `bun run test:small`       | 49テスト成功、Statements 97.41%                                          |
| `bun run test:coverage:pr` | 成功、Statements 97.41%、Branches 97.43%、Functions 94.11%、Lines 97.29% |
| `bun run test:storybook`   | 23テスト成功                                                             |
| `bun run build`            | 成功                                                                     |

Storybookのアクセシビリティ検証では、エラー要素の通知とAlertのコントラストに関する失敗を修正後、10ファイル・23テストが成功した。

## 注意点

- `aria-errormessage`だけでは動的な更新通知の仕組みにならないため、エラー要素側のライブリージョン設計も必要になる。
- `aria-live="polite"`は通常の入力エラー向けであり、緊急性の高い通知へ一律に使うものではない。
- 自動アクセシビリティ検査の成功は、スクリーンリーダー、キーボード、ズームを含むWCAG適合の証明ではない。
- 今回の確認はStorybookと自動テストが中心で、実機のスクリーンリーダー検証は未実施である。

## まとめ

フォームエラーは、表示・状態・関連付け・通知を分けて設計すると検証しやすい。`aria-errormessage`で対象を明示し、`aria-live`で更新を通知し、色はトークンとコントラスト基準で確認する。これらをコンポーネント利用側の振る舞いテストとStorybookのa11y検証で回帰から守る。

## 参考資料

- [MDN: `aria-errormessage`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-errormessage)
- [MDN: Color contrast](https://developer.mozilla.org/en-US/docs/Web/Accessibility/Guides/Understanding_WCAG/Perceivable/Color_contrast)
- `mojica` commit `4bc87be fix: announce field errors to assistive technology`
- `mojica` commit `0d900cd fix: improve alert description contrast`
