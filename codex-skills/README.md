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
- [StorybookのCanvas余白と本番レスポンシブレイアウトを切り分ける](./23-storybook-canvas-and-responsive-vrt-scope.md)

## 共通の前提

- Windows 11
- PowerShell 7.6.4およびWindows PowerShell 5.1
- 2026年8月11日時点のCodex CLIインストーラーとローカルSkill構成

Codex CLIやSkillの仕様変更後は、公式資料と検証コマンドを再確認する必要がある。
