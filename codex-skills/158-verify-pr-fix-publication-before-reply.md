# PRの「修正済み」をコミット・反映・返信・解決に分けて確認する

## はじめに

PRの指摘へ「修正しました」と返信する際、ローカルの変更だけではPRに反映されたことを説明できませんでした。今回はコミット、リモートのhead、元のThreadへの返信、解決済み状態を順に照合しました。

## 前提・環境

2026年10月22日、blog-wordcloudのPR #35で行った対応です。対象の指摘は、frontend CIへ本番buildを追加するという1件でした。GitHubの取得・投稿・Thread解決にはGitHub MCPを使用しました。

## やってみた結果

ユーザーの修正を検証し、残るコンパイルエラーを最小修正した後、次の3コミットをPRの作業ブランチへ反映しました。

| コミット | 目的 |
| --- | --- |
| 997ed8b | 存在しない画像参照を削除 |
| 90bfa0a | storiesの未使用React importを削除 |
| 7fee2d2 | CIへ本番ビルドを追加 |

この後で元のレビューThreadへ返信し、Threadを解決済みにしました。レビュー対象外のBot通知へ重複返信は行っていません。

## 実装・検証

### 返信先と作業対象のheadを特定する

Threadと全返信、Review本文、Conversationコメント、PRのbase/headを取得しました。未解決状態を平坦なコメント一覧から推定しないよう、Threadのresolved情報を確認しました。

開始時はGitHubのPR headとローカルHEADが一致していました。作業ブランチ名が一致するだけで、別のheadへ修正を加えることはしませんでした。

### 修正を反映してから「修正済み」と書く

本番build、型検査、Storybookの8テストが成功したことを確認してコミットしました。作業ブランチを通常pushした後、git ls-remoteとGitHub MCPから取得したPR headが7fee2d2の完全SHAと一致することを確認しました。

返信にはCI追加のコミット、変更内容、成功した検証を含めました。同時に、既存のLint違反が残っていることも記載しました。全CI成功とは書いていません。

### 返信後にThreadの状態を確認する

[元Threadへの返信](https://github.com/kishimin/blog-wordcloud/pull/35#discussion_r4348149857)を投稿した後、同じThreadを解決しました。再取得した本文とis_resolved=trueを確認しました。

対象指摘に対して必要だったBuild stepはGitHub Actionsでも成功しました。ただしFrontend checksは別のESLint違反で失敗しています。「本番buildの検査追加に対応した」と「PRの全品質検査に合格した」は分けて記録しました。

## つまずいたところ

ローカルbuildを試すと、ユーザーの修正後にもHeaderの未使用React importが残っていました。そのまま「ビルドできる」とは返信せず、失敗を再現して除去し、成功後に反映しました。

## 学んだこと

対応済みの主張には対象が必要でした。今回確認した対象は、修正コミットが存在する場所、PR headへ反映された内容、返信したThread、解決状態の4つです。全CIの結果が別に残っていても、何を直して何が未完了なのかを具体的に説明できました。

## 根拠と制約

根拠は3コミット、PR headの照合、返信本文とThreadの再取得、[Frontend CIの実行結果](https://github.com/kishimin/blog-wordcloud/actions/runs/39776488863)です。これは今回の指摘への対応記録であり、未解決の仕様判断や修正未反映のThreadまで解決する手順ではありません。

既存返信の英訳・編集そのものを扱う[以前の記事](./92-edit-existing-pr-replies-in-english.md)とは対象を分けています。PRのマージは行っていません。
