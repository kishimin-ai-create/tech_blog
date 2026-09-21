# `everything-claude-code`をClaude Codeのユーザープラグインとして導入する

## はじめに

Claude Codeでエージェント、Skill、コマンド、フックをまとめて利用するため、`everything-claude-code`をMarketplace経由でユーザースコープへ導入しました。

この記事では、指定したGitHubリポジトリをClaude CodeのMarketplaceへ登録し、プラグインをインストールして有効化状態を確認する手順を記録します。

## 前提・環境

- OS: Windows
- Claude Code: 2.1.220
- 対象: [worldflowai/everything-claude-code](https://github.com/worldflowai/everything-claude-code)
- インストール範囲: user

## やってみた結果

Marketplaceの追加とプラグインのインストールが成功し、`everything-claude-code@everything-claude-code`は`user`スコープで`enabled`になりました。CLIの詳細表示では、Skills 26、Agents 9、Hooks 6、MCP servers 0と確認できました。

## 実装・検証

### Step 1：Marketplaceを追加する

Claude Codeのプラグイン管理CLIで、指定されたGitHubリポジトリをユーザースコープのMarketplaceとして登録しました。

```powershell
claude plugin marketplace add worldflowai/everything-claude-code --scope user
```

CLIはリポジトリを取得し、Marketplaceを検証したうえで、`Successfully added marketplace: everything-claude-code`を出力しました。

### Step 2：プラグインをインストールする

Marketplace名とプラグイン名を明示してインストールしました。

```powershell
claude plugin install everything-claude-code@everything-claude-code --scope user
```

実行結果は`Successfully installed plugin: everything-claude-code@everything-claude-code (scope: user)`でした。

### Step 3：有効化状態と構成を確認する

```powershell
claude plugin marketplace list
claude plugin list
claude plugin details everything-claude-code@everything-claude-code
```

Marketplace一覧では、次の登録を確認しました。

| 項目 | 結果 |
| --- | --- |
| Marketplace | `everything-claude-code` |
| Source | `worldflowai/everything-claude-code` |
| Plugin | `everything-claude-code@everything-claude-code` |
| Scope | `user` |
| Status | enabled |
| MCP servers | 0 |

詳細表示のProjected token costはAlways-onで約1,714 tokenでした。すべてのMCPを追加する操作は行っていません。

## つまずいたところ

今回のインストールではエラーは発生しませんでした。Marketplaceの取得、検証、プラグインの有効化をそれぞれCLIの出力で確認しました。

## 学んだこと

Claude Codeのプラグインは、構成ファイルを手動で個別コピーする方法もありますが、Marketplaceとプラグイン管理CLIを使うと、登録・インストール・有効化状態をClaude Code自身に管理させられます。MCP設定は別の構成要素なので、プラグインを導入しただけではMCPサーバーは追加されません。

## 制約・次の課題

- この確認では、導入した各SkillやAgentを実際に呼び出すところまでは検証していません。
- プラグインはAlways-onのコンテキストコストがあるため、利用しない構成を追加で有効化する場合はトークン使用量を確認する必要があります。
- MCPサーバーを使う場合は、対象サービスごとの認証情報と設定を別途確認する必要があります。

## まとめ

`claude plugin marketplace add`で`worldflowai/everything-claude-code`をMarketplaceへ登録し、`claude plugin install`でユーザースコープへ導入しました。`claude plugin list`で有効化済みであること、`claude plugin details`で構成要素を確認できました。

## 参考資料

- [everything-claude-code README](https://github.com/worldflowai/everything-claude-code)

確認日: 2026-09-21
