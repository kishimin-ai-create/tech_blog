# Codex Skills 技術記事

WindowsでのCodex CLI運用と、Codex・Claude Code間でユーザーSkillを設計・共有した際の知見をまとめる。各記事の確認日は本文末尾に記載する。

## インストールとトラブルシュート

- [Codex CLIのインストールで`Get-FileHash`が見つからないときの切り分け](./01-codex-get-filehash-install-error.md)

## Skill設計と共有

- [xUnitのテストケースをコードではなくコメントだけで設計するSkill](./02-comment-only-xunit-skill-design.md)
- [Claude CodeだけにあるSkillをCodexでも使えるようにする](./03-share-claude-skills-with-codex.md)
- [Skillを別のPCへ移植するためのパス監査を作る](./04-portable-skill-paths-audit.md)

## 共通の前提

- Windows 11
- PowerShell 7.6.4およびWindows PowerShell 5.1
- 2026年8月11日時点のCodex CLIインストーラーとローカルSkill構成

Codex CLIやSkillの仕様変更後は、公式資料と検証コマンドを再確認する必要がある。
