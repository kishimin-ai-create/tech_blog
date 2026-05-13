# CodeReviewAgent 完了後に ReviewResponseAgent を自動起動するルールを追加した

## 対象読者

- GitHub Copilot エージェントを活用してレビューフローを自動化したいエンジニア
- `copilot-instructions.md` でエージェント間の連携ルールを整備している人

---

## 背景

このリポジトリでは `CodeReviewAgent` がレビューファイルを `review/` に保存した後、`ReviewResponseAgent` を手動で呼び出してレビュー返信を書いていた。

手動起動には2つの問題があった。

1. **呼び出し忘れ** — コードレビューが完了しても返信ドラフトが作られないまま次の作業に進んでしまう
2. **返信の書き先が曖昧** — `ReviewResponseAgent` が返信テキストをエージェントの応答として出力するだけで、レビューファイル自体に書き込まれないことがあった

---

## 変更内容

### `copilot-instructions.md` — 自動起動ルールの追加

```markdown
# Post-task automation

- After fixing an error or resolving one error root cause, automatically invoke
  `@ArticleWriterAgent` and create one error-focused article for that fix.
- After `@CodeReviewAgent` completes and saves its review file, automatically
  invoke `@ReviewResponseAgent` with the newly created review file as context
  before proceeding to any other post-task step.
- After completing an implementation, fix, review-response, refactor, or similar
  repository task, automatically invoke post-task agents in this order:
  1. `@ArticleWriterAgent`
  2. `@WorkSummaryAgent`
- Run the post-task agents only after the main task changes are finished.
```

追加したのは2行目のルール。`@CodeReviewAgent` がレビューファイルを保存したら、次のタスクに移る前に `@ReviewResponseAgent` を自動で起動する。

### `ReviewResponseAgent.agent.md` — 返信の書き先を明示

返信テキストをエージェント応答として出力するだけでなく、**レビューファイル本体に直接書き込む**ことを Output と Definition of Done の両方に明記した。

```markdown
## Output

ReviewResponseAgent MUST deliver:

2. **Replies written directly into the review file** — append each reply text
   immediately after the corresponding `Useful? React with 👍 / 👎.` line in the
   review file itself. Do NOT produce replies as agent output text only.
```

```markdown
## Definition of done

- **Reply text is written directly into the review file** below each
  `Useful? React with 👍 / 👎.` line — the review file is the single source of
  truth for both findings and replies
```

返信フォーマットも具体化した。`Useful? React with 👍 / 👎.` の直後に返信を挿入し、`---` セクション区切りの前に置く。

```markdown
Useful? React with 👍 / 👎.

ご指摘ありがとうございます。{何をしたか、またはなぜ変更不要かを具体的に}

---
```

---

## この変更で解決できること

| 問題 | 解決策 |
|------|--------|
| CodeReview 後に ReviewResponse を呼び忘れる | `copilot-instructions.md` の自動起動ルールで強制 |
| 返信がエージェント応答にしか残らない | `ReviewResponseAgent.agent.md` にレビューファイルへの書き込みを明記 |
| 返信の挿入位置が不明確 | `Useful?` 行の直後に書き込む形式を具体例付きで規定 |

---

## まとめ

エージェント定義ファイルはコードと同じく「仕様書」として機能する。曖昧な出力先・曖昧な起動タイミングはエージェントの動作にそのまま反映される。

今回の変更のように「いつ起動するか（`copilot-instructions.md`）」と「何をどこに出力するか（`ReviewResponseAgent.agent.md`）」を両方のファイルで明示することで、エージェントの動作を確定させることができた。
