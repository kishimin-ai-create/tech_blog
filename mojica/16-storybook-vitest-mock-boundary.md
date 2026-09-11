# Storybookのブラウザー実行でVitestの`vi.fn`を使わない

## 要約

Storybookのブラウザー上で実行するStoryにVitestの`vi.fn`をimportすると、Vitest内部状態が存在せず`customEqualityTesters`参照エラーになることがある。Storyのactionには`storybook/test`の`fn`を使い、VitestのmockはNode側のテストに限定する。

## 発生した症状

Storybookテストで次のエラーが発生した。

```text
Cannot read properties of undefined (reading 'customEqualityTesters')
```

スタックトレースはStorybookのブラウザー依存へbundlingされたVitestのexpect拡張処理を指していた。

## 修正

Storyでは次のようにactionを定義した。

```ts
import { fn } from "storybook/test";

export const Default = {
  args: { onChange: fn() },
};
```

Vitestの`vi.fn`はSmallテストなどVitest実行環境だけで使用する。StorybookのPreviewにはアプリケーションProviderをdecoratorで設定し、各Storyから重複したProviderも除去した。

## 検証

`bun run test:storybook`で10ファイル・23テスト、`bun run build-storybook`で本番Storybookビルドが成功した。関連コミットは`cd84318`、`8f00a89`、`b4e1e9c`である。

## 学び

mock関数は名前が似ていても実行環境の境界が異なる。StorybookのStoryはStorybookが提供するtest utilitiesを使い、Vitest専用APIをブラウザーへ持ち込まないことが安全である。
