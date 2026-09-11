# Codex Skills 技術記事

WindowsでのCodex CLI運用と、Codex・Claude Code間でユーザーSkillを設計・共有した際の知見をまとめる。各記事の確認日は本文末尾に記載する。

## インストールとトラブルシュート

- [Codex CLIのインストールで`Get-FileHash`が見つからないときの切り分け](./01-codex-get-filehash-install-error.md)

## Skill設計と共有

- [xUnitのテストケースをコードではなくコメントだけで設計するSkill](./02-comment-only-xunit-skill-design.md)
- [Claude CodeだけにあるSkillをCodexでも使えるようにする](./03-share-claude-skills-with-codex.md)

## VS Codeワークスペース

- [VS Codeの既存ウィンドウへフォルダーを追加・削除する](./07-vscode-multi-root-workspace-cli.md)

## API・UI設計

- [ランダム郵便番号APIで「CSVの行」ではなく「一意な郵便番号」を抽選する理由](./08-random-postal-code-selection-unit.md)
- [地図と広告の障害を主要UIから分離する状態設計](./09-isolate-optional-ui-services.md)

## 共通の前提

- Windows 11
- PowerShell 7.6.4およびWindows PowerShell 5.1
- 2026年8月11日時点のCodex CLIインストーラーとローカルSkill構成

Codex CLIやSkillの仕様変更後は、公式資料と検証コマンドを再確認する必要がある。
