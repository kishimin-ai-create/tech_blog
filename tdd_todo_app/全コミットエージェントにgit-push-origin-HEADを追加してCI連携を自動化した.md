# 全コミットエージェントに `git push origin HEAD` を追加して CI 連携を自動化した

## 対象読者

- GitHub Copilot のカスタムエージェント（`.agent.md`）を複数運用しているチーム
- エージェントがコミットを作成しても CI が動かない問題に気づいている人
- マルチエージェント構成で「commit は自動なのに push は手動」という状態を解消したい人

---

## 問題の背景

`tddTodoApp` リポジトリには 13 個のカスタムエージェントが存在し、うち 11 個（FixAgent、GreenAgent、RedAgent など）はコードを変更したあとに `git commit` を実行する。

しかし、コミットしても `git push` が実行されなければ **GitHub Actions の CI ワークフローは一切起動しない**。結果として次のような状態が繰り返されていた。

```
エージェントが作業完了 → git commit は実行 → ローカルに留まる
          ↓
開発者が手動で git push → 初めて CI が起動
          ↓
「エージェントが自律的に動いている」はずが、CI 起動まで人手が必要
```

エージェントの自律動作フロー（Red → Green → Refactor → Review → ReviewResponse）において、CI の起動が人手に依存するのは設計の一貫性を欠く。

---

## 変更内容（コミット `cd88f3d`）

コミット権限を持つ 11 エージェント全員の `.agent.md` に `git push origin HEAD` を追加した。

### 対象エージェント一覧

| エージェント | 変更の性質 |
|---|---|
| FixAgent | 既存の commit 行の直後に `git push` 追加 |
| RefactorAgent | 既存の commit 行の直後に `git push` 追加 |
| StorybookCreatorAgent | 既存の commit 行の直後に `git push` 追加 |
| GreenAgent | 新規「Git Commit & Push」セクションを追加 |
| RedAgent | 新規「Git Commit & Push」セクションを追加 |
| ReviewResponseAgent | 新規「Git Commit & Push」セクションを追加 |
| CodeReviewAgent | 新規「Git Commit & Push」セクションを追加 |
| UIDesignAgent | 新規「Git Commit & Push」セクションを追加 |
| OpenApiWriterAgent | 新規「Git Commit & Push」セクションを追加 |
| PullRequestWriterAgent | 新規「Git Commit & Push」セクションを追加 |
| RegressionTestAgent | 新規「Git Commit & Push」セクションを追加 |

### 変更パターン A：既存 commit 行への追記（FixAgent の例）

```diff
 git add -A
 git commit -m "fix: <short description of what was wrong and what was corrected>"
+git push origin HEAD
```

FixAgent・RefactorAgent・StorybookCreatorAgent はすでに commit の手順が定義されており、その直後に 1 行追加するだけで済んだ。

### 変更パターン B：新規セクション追加（GreenAgent の例）

GreenAgent などは commit 手順自体が未定義だったため、「Git Commit & Push」という独立したセクションを追加した。

```markdown
## 📥 Git Commit & Push

After all tests pass and implementation is complete, commit and push:

```bash
git add -A
git commit -m "feat: <description of implemented feature>

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
git push origin HEAD
```
```

---

## 意図的に変更しなかったエージェント

| エージェント | 理由 |
|---|---|
| ArticleWriterAgent | 別コミットで **git commit/push 禁止** が明示されている記録専用エージェント |
| WorkSummaryAgent | 同上。diary/ にファイルを書くだけで git 履歴を変更してはならない |

これら 2 エージェントは `Prohibited Actions` セクションに以下のルールが入っており、push を追加することは禁止ルールと矛盾する。

```markdown
❌ Running `git commit`, `git push`, or any command that writes to git history —
   ArticleWriterAgent has **read-only** git access.
   File creation/editing is allowed; committing is not.
```

記録エージェントと実装エージェントの区別を維持することが、この変更全体の設計原則だ。

---

## この変更が解決すること

### Before: コミットまでが自動、CI 起動は手動

```
エージェントが実装 → git commit（自動）→ ローカル止まり
                                         ↓（手動 push が必要）
                                    GitHub Actions 起動
```

### After: コミットから CI 起動まで完全自動

```
エージェントが実装 → git commit → git push（自動）→ GitHub Actions 起動
```

GreenAgent がテストを通す実装をコミットしたら、その場で CI が走る。RefactorAgent がリファクタを完了したら即座にすべてのワークフローが検証を始める。エージェントの自律動作フロー全体が、CI を含めて完結するようになった。

---

## 注意点

### `git push origin HEAD` を使う理由

`git push` だけでも動くケースが多いが、`origin HEAD` を明示することで：

- カレントブランチが `origin` に対して追跡設定されていない状況でも確実に動く
- どのリモート・どのブランチに push するかが `.agent.md` のコードから一目で分かる

エージェントが動くブランチは `frontend` や `main` と固定されておらず、作業ブランチが変わるたびに push 先を手動で調整するのは非現実的なため、`HEAD` での相対指定を採用している。

### 権限の前提

`git push` が成功するには、GitHub Actions の runner またはローカル環境が当該リポジトリへの write 権限を持っている必要がある。Copilot エージェントが GitHub 上で動く場合は `GITHUB_TOKEN` の permissions 設定（`contents: write`）が別途必要になることがある。

---

## まとめ

- 11 のコミットエージェントに `git push origin HEAD` を追加し、コミットから CI 起動までを完全自動化した
- 記録専用の ArticleWriterAgent・WorkSummaryAgent は意図的に変更せず、「read-only git アクセス」の原則を維持した
- `origin HEAD` の明示により、ブランチ追跡設定に依存しない安定した push が実現する
- この変更でエージェントの自律動作フローが CI を含めてエンドツーエンドで閉じるようになった
