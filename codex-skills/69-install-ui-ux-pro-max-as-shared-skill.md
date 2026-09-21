# `ui-ux-pro-max`を共有Skillとして導入する

## はじめに

CodexとClaude Codeの両方でUI/UX作業向けのSkillを使うため、GitHubで公開されている`ui-ux-pro-max-skill`をユーザー共通のSkillとして導入しました。

この記事では、公式CLIを使って`$HOME/.agents/skills`へインストールし、Skillファイルが生成されたことを確認する手順を記録します。

## 前提・環境

- OS: Windows
- Node.js: v26.7.0
- npm: 12.0.2
- 対象: [nextlevelbuilder/ui-ux-pro-max-skill](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill)
- 導入先: `$HOME/.agents/skills/ui-ux-pro-max`

## やってみた結果

`ui-ux-pro-max-cli`をグローバルインストールし、`universal`ターゲットで初期化しました。`$HOME/.agents/skills/ui-ux-pro-max/SKILL.md`が作成され、配下には68ファイルが存在しました。

この配置はプロジェクト内ではなくユーザー共通のSkill置き場なので、複数のAIコーディングアシスタントから同じSkillを参照できます。

## 実装・検証

### Step 1：CLIをインストールする

リポジトリのREADMEに記載されたパッケージをグローバルインストールしました。

```powershell
npm install -g ui-ux-pro-max-cli
```

実行結果は`added 23 packages in 14s`でした。

### Step 2：Universal向けにグローバル初期化する

Claude Code専用のプロジェクト設定ではなく、共有Skill形式で導入するため、`universal`と`--global`を指定しました。

```powershell
uipro init --ai universal --global
```

CLIは`Universal (.agents/skills/) (global)`として処理し、`UI/UX Pro Max installed successfully!`を出力しました。

### Step 3：生成物を確認する

```powershell
$skill = Join-Path $HOME '.agents\skills\ui-ux-pro-max\SKILL.md'
Test-Path -LiteralPath $skill
Get-ChildItem (Split-Path $skill) -Recurse -File | Measure-Object
```

今回の確認では、`SKILL.md`の存在が`True`になり、配下のファイル数は68でした。後続の確認でも`SKILL.md`のサイズは28,066 bytesでした。

## つまずいたところ

今回の導入ではエラーは発生しませんでした。CLIの出力と生成先のファイル確認によって、インストール先を検証しました。

## 学んだこと

`ui-ux-pro-max`は、プロジェクトへ個別コピーするよりも、CLIの`universal --global`を使うことで共有Skillとして導入できます。AIアシスタント向けの対象を指定するCLIでは、導入先の責務に合わせて`universal`と`--global`を明示することが重要です。

## 制約・次の課題

- この確認では、実際のUI実装タスクを依頼してSkillの発火結果までは検証していません。
- CLIやSkillの更新後は、生成先と`SKILL.md`の存在を再確認する必要があります。
- 既存の共有Skillと同名のSkillを導入する場合は、上書きや更新経路を事前に確認する必要があります。

## まとめ

`ui-ux-pro-max-cli`をグローバルインストールし、`uipro init --ai universal --global`を実行することで、`ui-ux-pro-max`を`$HOME/.agents/skills`へ導入できました。ファイルの存在と生成数を確認したため、共有Skillとして利用できる状態です。

## 参考資料

- [ui-ux-pro-max-skill README](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill)

確認日: 2026-09-21
