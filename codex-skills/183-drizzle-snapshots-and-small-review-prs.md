# 小さなスキーマ変更でも生成スナップショットが差分を増やす

レビュー練習のため、profile-hubでは目的を絞ったPRを積み上げている。ところがテーブルを一つずつ追加しても、Drizzleの生成するスキーマスナップショットにはそれまでの構造が含まれるため、差分行数は増えた。

Skillの [PR #58](https://github.com/kishimin-ai-create/profile-hub/pull/58) は追加162行、次のFavouriteの #59 は266行、SocialLinkの #60 は381行、Portfolioの #61 は485行、Profileの #62 は520行、Userの [#63](https://github.com/kishimin-ai-create/profile-hub/pull/63) は583行だった。各PRの目的は絞られているが、行数だけを見ればレビュー負荷は小さくない。

レビューでは、手書きのスキーマとSQLの意味を読み、生成スナップショットがその変更に対応しているかも確認する。生成物だからといって無視してよいという結論ではない。生成物の検証方法や見せ方は今後の改善課題であり、この時点では負担が解消されたとは言えない。

PRの依存関係を追う方法は[積み上げPRの記事](./172-small-stacked-prs-for-review-practice.md)にまとめた。ここでは、変更目的の小ささと表示される差分量が一致しない点を扱った。
