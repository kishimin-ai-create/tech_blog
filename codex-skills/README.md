# Codex Skills 技術記事

WindowsでのCodex CLI運用と、Codex・Claude Code間でユーザーSkillを設計・共有した際の知見をまとめる。各記事の確認日は本文末尾に記載する。

## インストールとトラブルシュート

- [Codex CLIのインストールで`Get-FileHash`が見つからないときの切り分け](./01-codex-get-filehash-install-error.md)

## Skill設計と共有

- [xUnitのテストケースをコードではなくコメントだけで設計するSkill](./02-comment-only-xunit-skill-design.md)
- [Claude CodeだけにあるSkillをCodexでも使えるようにする](./03-share-claude-skills-with-codex.md)

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

## 共通の前提

- Windows 11
- PowerShell 7.6.4およびWindows PowerShell 5.1
- 2026年8月11日時点のCodex CLIインストーラーとローカルSkill構成

Codex CLIやSkillの仕様変更後は、公式資料と検証コマンドを再確認する必要がある。
