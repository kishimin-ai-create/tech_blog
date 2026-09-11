# Codex Skills 技術記事

WindowsでのCodex CLI運用と、Codex・Claude Code間でユーザーSkillを設計・共有した際の知見をまとめる。各記事の確認日は本文末尾に記載する。

## インストールとトラブルシュート

- [Codex CLIのインストールで`Get-FileHash`が見つからないときの切り分け](./01-codex-get-filehash-install-error.md)

## Skill設計と共有

- [xUnitのテストケースをコードではなくコメントだけで設計するSkill](./02-comment-only-xunit-skill-design.md)
- [Claude CodeだけにあるSkillをCodexでも使えるようにする](./03-share-claude-skills-with-codex.md)
- [Skillを別のPCへ移植するためのパス監査を作る](./04-portable-skill-paths-audit.md)

## React Hook Formとテスト設計

- [React Hook Formを機能専用Hookへ切り出す境界を決める](./05-react-hook-form-custom-hook-boundary.md)
- [React Hook Formの専用Hookをどうテストするか](./06-react-hook-form-hook-test-assertions.md)
- [Codex CLI更新時の配布元フォールバック警告を抑制する](./07-codex-installer-release-source.md)
- [Retry-Afterのカウントダウンをコールバック回数ではなく時刻で計算する](./08-retry-countdown-wall-clock.md)
- [ImageGenerationFormの統合テストをSmallから設計する](./09-image-generation-form-test-plan.md)
- [Figmaの画面仕様をReactのレイアウトへ落とし込む](./10-figma-ui-spec-to-react-layout.md)
- [モバイルの横スクロールを`min-width`から切り分ける](./11-mobile-horizontal-overflow-min-width.md)
- [ImageGenerationFormの状態をStorybookで再現可能にする](./12-image-generation-storybook-states.md)
- [Zodで画像生成フォームのHEXカラー入力を検証する](./13-hex-color-validation-with-zod.md)
- [ReactフォームのPNG自動ダウンロードをブラウザAPI境界でテストする](./14-testing-browser-download-boundary.md)
- [Playwrightのfixtureと関数形式Page ObjectでE2Eの責務を分離する](./24-playwright-fixture-function-pom.md)
- [実APIの画像生成E2EでEnter送信を検証する境界を決める](./25-real-api-keyboard-e2e-boundary.md)
- [ErrorFallbackの日本語・英語StoryをPlaywright VRTで比較する](./26-error-fallback-storybook-vrt-locales.md)
- [実APIの画像生成E2Eで並列実行とSafariダウンロードを切り分ける](./27-e2e-api-rate-limit-and-safari-download-timeout.md)
- [Page Objectの関数参照とlocale Selector対応表をLintで守る](./28-eslint-page-object-and-locale-selector-rules.md)
- [StorybookのCanvas余白と本番レスポンシブレイアウトを切り分ける](./23-storybook-canvas-and-responsive-vrt-scope.md)
- [Playwright E2Eを既存のテストサイズ別CIへ統合する](./29-integrate-playwright-e2e-into-sized-ci.md)
- [CIのE2Eダウンロードがタイムアウトした原因をAPIレート制限から切り分ける](./31-ci-e2e-download-timeout-rate-limit.md)
- [認証なしでMojica APIとGlyph Forgeのレート制限を分担する](./32-anonymous-api-rate-limit-boundary.md)

## VS Codeワークスペース

- [VS Codeの既存ウィンドウへフォルダーを追加・削除する](./07-vscode-multi-root-workspace-cli.md)

## API・UI設計

- [ランダム郵便番号APIで「CSVの行」ではなく「一意な郵便番号」を抽選する理由](./08-random-postal-code-selection-unit.md)
- [地図と広告の障害を主要UIから分離する状態設計](./09-isolate-optional-ui-services.md)

## デプロイと運用

- [5並列E2E実行とAPIレート制限を両立する](./30-run-five-e2e-workers-with-a-bounded-api-rate-limit.md)
- [PlaywrightカバレッジをCIの品質ゲートにしなかった理由](./31-why-playwright-coverage-is-not-a-ci-quality-gate-yet.md)
- [Cloudflare PagesでBunプロジェクトの自動npm installが失敗したときの切り分け](./32-cloudflare-pages-bun-lockfile-npm-install-failure.md)
- [さくらのクラウドAppRunでコンテナレジストリ認証に失敗したときの切り分け](./33-sakura-apprun-registry-authentication.md)
- [ViteのFrontendをCloudflare Pages、APIをさくらのクラウドAppRunへ分ける判断](./34-cloudflare-pages-and-sakura-apprun.md)
- [本番E2EでCORS再デプロイと5ブラウザーの画像生成を確認する](./36-validate-production-e2e-after-cors-and-parallelism.md)
- [Blob URLと`download`属性がアプリ内ブラウザーで保証されない理由](./37-blob-download-limitations-in-in-app-browsers.md)
- [UI変更後にLinux用Visual Regressionベースラインを更新する](./38-update-linux-visual-baselines-after-ui-change.md)
- [生成画像の種類をダウンロードファイル名で検証する](./39-verify-image-type-in-download-filename.md)
- [実ファイルを根拠にモノレポのREADMEを整備する](./40-build-an-evidence-based-monorepo-readme.md)
- [Bunを残したままHono APIをCloudflare Workersへ移す設計](./41-run-bun-hono-on-workers-with-supabase.md)
- [Wrangler生成Runtime型がBunの型チェックを壊した原因と分離方法](./42-separate-wrangler-runtime-types-from-bun.md)
- [Supabase Direct connectionでENOTFOUNDになったときのmigration経路](./43-use-supabase-session-pooler-for-local-migrations.md)
- [VercelがReadyでも404になるときはFramework Presetを確認する](./44-fix-vercel-ready-deployment-404-with-nextjs-preset.md)

## リリースE2Eとコンテナ容量

- [リリースE2Eのflakyを同時実行数から切り分ける](./04-release-e2e-flaky-concurrency.md)
- [AppRunとGlyph Forgeの容量設定を分けて考える](./05-apprun-glyph-forge-capacity.md)

## 共通の前提

- Windows 11
- PowerShell 7.6.4およびWindows PowerShell 5.1
- 2026年8月11日時点のCodex CLIインストーラーとローカルSkill構成

Codex CLIやSkillの仕様変更後は、公式資料と検証コマンドを再確認する必要がある。
