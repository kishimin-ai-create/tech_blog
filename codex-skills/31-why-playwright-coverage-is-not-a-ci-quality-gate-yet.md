# PlaywrightのカバレッジをCIの品質ゲートにする前に確認すること

## 対象読者

Playwright E2Eの実行結果にコードカバレッジを加え、CIの品質ゲートとして運用できるか検討しているフロントエンド開発者を対象にする。

## スコープ

Playwright Coverage API、ChromiumのV8カバレッジ、Mojicaのブラウザー構成を比較し、今回の調査で品質ゲート化を見送った理由を説明する。カバレッジの具体的な閾値設計や、各ブラウザーの内部実装の性能比較は扱わない。

## 結論

Playwrightにはブラウザー内でCoverage APIを取得する方法があるが、今回のMojicaの5ブラウザー構成へそのまま適用して、全ブラウザー共通のCI品質ゲートにするのは適切ではないと判断した。調査した方法はChromiumのV8カバレッジを中心としたもので、WebKitやモバイルエミュレーションを含む現在の検証範囲を同じ条件で評価できないためである。

## 調査の背景

MojicaのPlaywright設定では、Google Chrome、Microsoft Edge、Desktop WebKit、Android Chrome、iPhone Safariをテストプロジェクトとして定義している。E2Eは実APIを使った画像生成、ダウンロード、画面遷移、StorybookのVRTを含むため、単にブラウザー操作の成功数だけでなく、コードの実行範囲を測定できるかを検討した。

一方、フロントエンドの既存カバレッジはVitestの`@vitest/coverage-v8`で収集している。E2Eカバレッジを追加する場合、既存の計測値と異なる収集方法・対象ブラウザー・統合方法を整理する必要がある。

## 調査した方法

PlaywrightのCoverage APIを使う場合、テスト中にページ上でカバレッジを開始し、操作後に停止して結果を取得する。ChromiumではV8由来のJavaScriptカバレッジを収集できる。この方法は「実際のブラウザー操作でどのコードが実行されたか」を確認する目的には使える。

しかし、今回の目的は5プロジェクトのE2Eを同じ品質ゲートで評価することである。Chromium向けのV8カバレッジだけを取得した場合、WebKitやiPhone SafariのE2Eが実行したコードは同じレポートへ反映されない。Chromiumだけの結果を全ブラウザーのカバレッジと扱うと、測定対象と結論が一致しない。

## Mojicaにそのまま導入しなかった理由

| 観点 | 調査結果 |
| --- | --- |
| 対象ブラウザー | 現在は5プロジェクト。Chromium系だけでは全体を表さない |
| 収集方式 | Playwright Coverage APIはブラウザー内で取得する追加処理が必要 |
| 既存計測との関係 | VitestのV8カバレッジと、E2Eのブラウザー実行カバレッジは目的とデータ形式が異なる |
| 品質ゲートへの適合 | 全プロジェクトを同じ条件で比較できず、今回のCIゲートには不向き |

この判断は、Coverage APIが利用できないという意味ではない。Chromium限定の補助的な調査や、特定フローが実行したコードを調べる用途には利用できる。今回見送ったのは、現在の5ブラウザー構成の総合カバレッジを表す単一の必須指標として扱うことである。

## 代わりに維持する検証

現在のCIでは、VitestのSmall・Medium・Large・Storybookテストでカバレッジを収集し、リポジトリで定義された指標を確認する。Playwrightはコードカバレッジではなく、実ブラウザーでのユーザー操作、実APIとの接続、ダウンロード、VRTを検証する責務を持つ。

この分担により、ユニット・コンポーネント側では測定可能なコード網羅率を品質ゲートにし、E2E側ではブラウザーごとの振る舞いを独立して検証できる。テスト層の目的が異なるため、E2Eの実行件数をVitestのカバレッジ閾値へ直接合算していない。

## 事実・判断・未確認事項

- FACT: `frontend/playwright.config.ts`は5つのブラウザー系プロジェクトを定義している。
- FACT: フロントエンドのVite設定では`@vitest/coverage-v8`を使っている。
- FACT: Playwright Coverage APIとChromiumのV8カバレッジを調査対象にした。
- INFERENCE: Chromiumの計測結果だけでは、WebKitを含む5プロジェクト全体の共通品質ゲートを表現できない。
- PROPOSAL: 将来E2Eカバレッジを導入する場合は、対象ブラウザー、計測形式、既存Vitestカバレッジとの統合単位を先に決める。
- ASSUMPTION: 今回の調査では、WebKitを含む全プロジェクトで同一形式のカバレッジを収集・統合する実装までは確認していない。

## トレードオフ・今後の課題

E2Eカバレッジを品質ゲートにしないため、E2Eが実行したコードの網羅率は現在のCIで数値化されない。一方で、測定条件が異なる値を1つの閾値へ混ぜることによる誤解を避けられる。将来、全対象ブラウザーで一貫した収集方法とレポート統合が用意できた時点で、品質ゲート化を再評価する。

## まとめ

PlaywrightのCoverage APIは有用な調査手段だが、利用可能であることと、CIの品質ゲートに適していることは別である。MojicaではChromiumのV8カバレッジだけでは5ブラウザー構成を代表できないため、現時点ではVitestのカバレッジとPlaywrightの振る舞い検証を分離する判断にした。

## 参考

- `mojica/frontend/playwright.config.ts`
- `mojica/frontend/vite.config.ts`
- `mojica/frontend/package.json`
- `mojica/.github/workflows/ci-push.yml`
- `mojica/.github/workflows/ci-pull-request.yml`
- `mojica/.github/workflows/ci-nightly.yml`

確認日: 2026-09-06
