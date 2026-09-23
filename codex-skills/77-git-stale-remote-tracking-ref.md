---
title: "リモートブランチ削除後にGitのpullとPublishが失敗する理由"
tags: [Git, GitHub, Pull Request, トラブルシュート]
---

# リモートブランチ削除後にGitのpullとPublishが失敗する理由

## 結論

ローカルブランチが存在しないリモートブランチを追跡していると、`git pull`はリモート参照を見つけられず、GUIのPublish表示も期待どおりにならない。リモート参照をpruneし、追跡先を解除してから、公開時に新しいリモートブランチを明示する。

## 症状

次のコマンドでエラーになった。

```text
git pull --tags origin feature/collect-by-source-entity-key
fatal: couldn't find remote ref feature/collect-by-source-entity-key
```

一方、ローカルの`git branch -vv`では、そのブランチが`origin/feature/collect-by-source-entity-key`を追跡しているように表示されていた。

## 切り分け

ローカル設定とリモートの実体を分けて確認する。

```powershell
git config --get-regexp "^branch\..*\.(remote|merge)$"
git ls-remote --heads origin feature/collect-by-source-entity-key
```

ローカルには追跡設定とremote-tracking refが残っていたが、`git ls-remote`には対象ブランチが返らなかった。reflogには過去のpushによる更新履歴が残っていたため、過去には存在したリモートブランチが削除され、ローカル情報だけが残った状態と判断できる。

## 解決方法

```powershell
git fetch --prune origin
git branch --unset-upstream
```

新しい作業ブランチを公開する場合は、対象ブランチを明示する。

```powershell
git push -u origin feature/example
```

`git push`が`main`へ送る可能性を避けるため、作業ブランチ作成直後に`git branch -vv`と`git config`で追跡先を確認する。

## 注意点

`git push --dry-run`は送信先の計算や差分を確認できるが、実際のリモート公開が完了した証拠ではない。公開後は`git ls-remote`やGitHub上のブランチ存在を確認する。

## まとめ

ローカルの追跡設定は、リモートブランチの現在状態を保証しない。`pull`の失敗とPublish非表示が同時に起きたら、追跡設定、remote-tracking ref、実リモートの3つを分けて確認する。
