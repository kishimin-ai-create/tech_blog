# エージェントの git コミット権限を廃止し Copilot の完全自律動作を強制した

## 対象読者

- GitHub Copilot のカスタムエージェント（`.agent.md`）を複数管理しているチーム
- エージェントが予期せず git 履歴を書き換える問題を防ぎたい人
- エージェントが確認プロンプトで作業を止めてしまうことに課題を感じている人

---

## 問題の背景

`tddTodoApp` リポジトリには 13 個のカスタムエージェントが存在する。そのうち **ArticleWriterAgent** と **WorkSummaryAgent** はそれぞれ `blog/` と `diary/` にファイルを書き込む記録・文書化エージェントだ。

これらのエージェントの役割は「ファイルを生成して保存する」ことであり、**git 履歴を変更する必要はない**。しかし変更前は、git commit / git push を実行することを明示的に禁止するルールが存在しなかった。

また、`copilot-instructions.md` の Autonomy（自律動作）セクションには「確認を求めないこと」というルールがあったが、**git 操作（commit / push）については明文化されていなかった**。その結果、エージェントが git commit の前に確認を取ったり、ReviewResponseAgent が「もう少し情報をください」と作業を止めるケースが発生していた。

これらは、AI エージェントによる自動化フローにおいて典型的な「過剰な人間介入要求」の問題だ。

---

## 変更内容

### 1. ArticleWriterAgent / WorkSummaryAgent への git 権限廃止

`ArticleWriterAgent.agent.md` と `WorkSummaryAgent.agent.md` の **Prohibited Actions** セクションに、以下の禁止ルールを追加した。

```markdown
❌ Running `git commit`, `git push`, or any command that writes to git history —
   ArticleWriterAgent has **read-only** git access.
   File creation/editing is allowed; committing is not.
```

WorkSummaryAgent にも同様の制約が設けられた。

| エージェント | git 読み取り | ファイル書き込み | git commit/push |
|---|---|---|---|
| ArticleWriterAgent | ✅ 許可 | ✅ `blog/` のみ | ❌ 禁止 |
| WorkSummaryAgent | ✅ 許可 | ✅ `diary/` のみ | ❌ 禁止 |

この区別には明確な理由がある。記録エージェントは **証拠収集（git log / diff の読み取り）** のために git 読み取り権限を必要とする。しかし、「コミットを作成する」という行為はメインタスクを担うエージェントや開発者自身が行うべきであり、記録エージェントが git 履歴を書き換えると作業ログの追跡が困難になる。

### 2. copilot-instructions.md Autonomy セクションの強化

変更前の Autonomy セクションは「確認や許可を求めるな」という原則を記載していたが、git 操作については例外的に曖昧だった。

変更後は **すべての操作カテゴリを明示列挙** し、git 操作を明確に含めた。

```markdown
# Autonomy

- **Never ask for confirmation or permission for any operation** — this includes
  git commits, file edits, file creation, deletions, running commands, applying
  fixes, or any other repository action. Receive the instruction and act immediately.
- Only ask a question when a fact is genuinely missing and makes correct
  implementation impossible.
- **Git operations** (commit, push, branch, rebase, etc.) require no
  confirmation. Execute them directly as part of the task.
```

変更のポイントは次の通り。

| 項目 | 変更前 | 変更後 |
|---|---|---|
| 確認禁止の対象 | 「操作全般」と曖昧な記述 | git commit / push を明示リスト |
| 質問を許可する条件 | 「必要な場合」 | 「実装を不可能にする事実が欠けている場合のみ」 |
| git 操作の扱い | 明示なし | 確認不要・即実行と明記 |

### 3. ReviewResponseAgent の「確認依頼」ブランチの削除

`ReviewResponseAgent.agent.md` には、レビューコメントの意図が不明な場合に「確認を求める」ブランチがあった。この分岐を削除し、**最も合理的な解釈を選んで進める** に統一した。

変更前のフロー:
```
レビューコメントを読む
  → 意図が明確なら修正
  → 意図が不明なら "確認を求める" ← これを削除
```

変更後のフロー:
```
レビューコメントを読む
  → 意図が明確なら修正
  → 意図が不明なら最も合理的な解釈で進める
```

ReviewResponseAgent が確認のために止まると、`CodeReviewAgent → ReviewResponseAgent` の自動連鎖パイプラインが中断してしまう。OrchestratorAgent が管理する自動フロー（Red → Green → Refactor → Review → ReviewResponse）はエージェントが止まらない前提で設計されているため、この変更は一貫性上も重要だった。

---

## 設計上の判断

### 「読み取り専用 git アクセス」という概念

記録エージェントに read-only git アクセスを与える設計は、**最小権限の原則** に基づく。

- エージェントが git log / diff を読む必要がある → 読み取りアクセスは必要
- エージェントがブログ記事・作業日誌を書く → ファイル書き込みは必要
- git 履歴を変更する必要は皆無 → commit / push は不要

この 3 点を切り分けることで、「記録エージェントが意図せず git 履歴を汚染する」リスクをルールレベルで排除できる。

### 「確認を求めるな」ルールをなぜ git に明示的に拡張したか

AI エージェントは「破壊的操作（git commit, git push など）はユーザー確認が必要」という暗黙の想定を持つ傾向がある。しかし、このリポジトリの開発フローでは：

1. エージェントは与えられた指示の範囲内で動く
2. 指示に git 操作が含まれている場合、それは明示的に許可されたものとして扱う
3. 確認プロンプトはフローを中断するため、生産性コストが高い

これを明示することで、エージェントが「commit してよいか？」と止まるパターンを根本から排除する。

---

## 注意点

- **ルールは `.agent.md` と `copilot-instructions.md` の両方を更新する必要がある** — エージェント個別のファイルにルールを書いても、`copilot-instructions.md` の Autonomy セクションが曖昧なままだと矛盾が生じる
- **「確認を求めない」は「事実が欠けていても進む」ではない** — 実装を不可能にする事実（例：APIキーの値、互換性のない 2 つの解釈）が欠けている場合は質問してよい。あくまで「許可を求める」行為を禁止している
- **ReviewResponseAgent の「最も合理的な解釈」は判断の質に依存する** — 複雑なレビューコメントで解釈が完全に二択に割れるケースは、エージェントの出力を確認して必要なら修正することが前提

---

## まとめ

今回の変更は、3 つの独立したルール修正で構成されている。

1. **ArticleWriterAgent / WorkSummaryAgent の git commit/push 禁止** — 記録エージェントが git 履歴を書き換えないよう、ルールで明示的に遮断
2. **Autonomy セクションへの git 操作の明示追加** — すべてのエージェントが git 操作を含むあらゆる操作で確認を求めない
3. **ReviewResponseAgent の確認依頼分岐の削除** — 自動連鎖パイプラインを止めない設計を維持

これらは個別の小さな変更だが、共通の目標に向かっている: **エージェントが人間に許可を求めずに自律的に動作し、かつ git 履歴への書き込み権限は必要なエージェントだけが持つ**。この 2 点のバランスが、マルチエージェントによる自動化フローの信頼性を高める。
