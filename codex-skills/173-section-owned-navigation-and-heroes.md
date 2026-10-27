# 二つの入口を持つサイトで、共通レイアウトと専用Heroの境界を決める

## 結論

profile-hubではヘッダー・本文領域・フッターを共有し、EngineeringとHobbyのHeroはそれぞれが所有する形にしました。見た目が似ていても、訪問者に伝える内容と移動先の契約が同じとは限りません。

2026年10月27日に公開サイト基盤として実装し、[PR #42](https://github.com/kishimin-ai-create/profile-hub/pull/42)と[PR #44](https://github.com/kishimin-ai-create/profile-hub/pull/44)へ分けました。両PRは確認時点で未マージです。

## 背景

公開サイトにはホーム、Engineering、Engineering Contact、Hobby、Hobby Contactの5ルートがあります。Figmaには各セクション専用の挨拶があり、共通のHelloを全ルートで使う案はIssueでも採用されていませんでした。

会話では共通ヘッダーにEngineeringとHobbyを並べる案を一度提示しました。しかし、ユーザーが「相互移動できない」と訂正し、ホームへのロゴリンクは残す形に確定しました。

## 採用した移動の契約

| 現在地                   | ヘッダーのリンク                       | 別セクションへの直接リンク |
| ------------------------ | -------------------------------------- | -------------------------- |
| EngineeringとそのContact | Engineering、Contact、ホームへ戻るロゴ | 置かない                   |
| HobbyとそのContact       | Hobby、Contact、ホームへ戻るロゴ       | 置かない                   |
| ホーム                   | 両セクションへの入口                   | 入口として置く             |

別セクションのURLを直接入力することまで制限したものではありません。認証・認可の境界ではなく、サイト内の導線の仕様です。

## コンポーネントの所有者を分ける

共通のSiteLayoutはルートに応じたナビゲーションと本文領域を組み立てます。Heroは次のように別のfeatureへ置きました。

```text
src/app/router/SiteLayout.tsx
src/features/engineering/components/EngineeringHero.tsx
src/features/hobby/components/HobbyHero.tsx
```

どちらのHeroも「こんにちは。」から始まりますが、その後の紹介文とセクション名は異なります。表示の骨組みに使うCSSは共有し、文言を持つHeroコンポーネントは共有していません。ホームもこれらのHeroを流用していません。

## 検証したこと

本番と同じルート定義を使うテストで、自セクションのContactへ進み、セクションへ戻り、ロゴからホームへ戻る動きを確認しました。ヘッダーに他方のリンクがないことも検証しています。

ブラウザでも両セクションのキーボードによるリンク実行を確認しました。ヘッダーの外観が共通でも、実際の行き先がセクションごとに違うことをテスト対象にしました。

## トレードオフ

似たJSXが二つのHeroに残ります。今回は実際の読み手とコピーの違いを優先しました。共通化を一般に否定する判断ではありません。将来、同じ責務のUIを別アプリも必要としたら、その時点で抽出範囲を検討します。

プロフィールやPortfolioの本文はこの基盤では実装していません。ここで確認したのは入口と導線の契約です。

## 根拠

- [本番のSiteLayout](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/src/app/router/SiteLayout.tsx)
- [ルーター境界のテスト](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/src/app/router/router.small.test.tsx)
- [Issue #16のHeroとナビゲーション要件](https://github.com/kishimin-ai-create/profile-hub/issues/16)
