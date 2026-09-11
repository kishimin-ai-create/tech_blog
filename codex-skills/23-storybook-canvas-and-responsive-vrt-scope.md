# StorybookのCanvas余白と本番レスポンシブレイアウトを切り分ける

## 対象読者

Storybookの表示を本番画面へ近づけるときに、Canvasの表示設定とReact側のレスポンシブ実装を混同したくないフロントエンド開発者を対象にする。

## 結論

StorybookのCanvas余白は、コンポーネントのレイアウト不具合とは別の表示レイヤーである。まず本番相当のReactレイアウトをブラウザ幅ごとに確認し、その後にStorybookの`layout`設定を調整する。VRT（Visual Regression Testing）は、Canvas設定を変更した場合も、意図した表示領域を含む基準として扱う範囲を明確にしてから更新する。

## 発生した問題

Mojicaの画像生成画面では、モバイル幅でPaperが画面幅を超えないことを確認するため、親コンテナ、左右余白、Paperの幅を段階的に調整した。変更履歴には`w-screen`の追加、`w-full`への変更、`max-width`のブレークポイント適用が含まれる。これらは本番レイアウトの問題を解く変更であり、Storybook Canvasの余白そのものを変更するものではない。

一方、Storybookを画面全体へ広げるために`parameters.layout = "fullscreen"`を追加すると、Storyの外側の見え方が変わる。最終的にはページStoryへ強制的なfullscreen設定を残さず、標準Canvasのレイアウトへ戻した。

## 切り分け方

確認対象を次の2層に分ける。

| 層 | 確認するもの | 変更箇所 |
| --- | --- | --- |
| 本番レイアウト | 390px、768px、1440pxでの親幅、左右余白、カード幅 | `ImageGenerationScreen.tsx`とCSSクラス |
| Storybook表示枠 | Canvasの外側余白、fullscreen指定、Story固有の表示条件 | `*.stories.tsx`のparameters |

本番レイアウトでは、モバイルのページ左右余白を維持しながらカードを親幅に収め、タブレット以上で最大幅を制限する。StorybookのCanvasをfullscreenにすることは、そのレイアウトを修正する代わりにはならない。

## VRTの範囲

VRTでは、何を基準画像へ含めるかを先に決める必要がある。実際のアプリケーションのページ全体を比較するなら、Header・Footer・ページ余白を含む統合境界が対象になる。コンポーネント単体の見た目を比較するなら、Canvasの標準余白を基準にするか、テスト専用の表示枠を用意してその枠を固定する。

Storybookの`layout`を全Storyへ適用する変更は、実装の見た目だけでなく基準画像の座標系も変更する。そのため、fullscreen化を採用する場合は既存VRTの一括更新ではなく、対象Storyと期待する表示領域を明記してから更新する。今回のブランチではVRT基準画像の更新は未実施である。

## 事実・判断・未確認事項

- FACT: `ImageGenerationScreen`のモバイル幅と最大幅を複数コミットで調整した。
- FACT: Storybookのfullscreen設定を一度追加したが、最終コミットで削除した。
- FACT: 実装プランは390px、768px、1440pxでのレスポンシブ確認とVRTを別の検証対象としている。
- INFERENCE: Canvasの表示枠と本番レイアウトを分離すると、CSS変更とStorybook設定変更の原因を切り分けやすい。
- ASSUMPTION: VRTの最終的な所有Playwrightプロジェクトと基準画像の更新手順は、E2E環境確定後に決める必要がある。

## まとめ

モバイルの横幅問題はReact側の親子レイアウトで解決し、Storybookの余白問題はCanvas設定として切り分ける。VRTは見た目の差分を自動検知するだけでなく、どの表示境界を正とするかを固定する仕組みなので、fullscreen化や基準画像更新の影響範囲を先に定義することが重要である。

## 参考

- `docs/v1/ui/design-tokens.md §6`
- `docs/v1/ui/component-design.md §4`
- `docs/v1/ui/branch-plans/test-mojica-ui-e2e.md`
- `frontend/src/features/image-generation/views/ImageGenerationScreen.tsx`
- `frontend/src/features/image-generation/views/ImageGenerationScreen.stories.tsx`
