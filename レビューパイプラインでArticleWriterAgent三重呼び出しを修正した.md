# レビューパイプラインでの ArticleWriterAgent・WorkSummaryAgent 三重呼び出しを修正した

## 対象読者

- GitHub Copilot のマルチエージェントパイプラインを設計・運用しているエンジニア
- エージェントチェーン（post-completion による自動連鎖）を構築していて、意図しない重複実行に悩んでいる人

---

## 問題の背景

このリポジトリでは、コードレビューから修正までを次のエージェントチェーンで自動化している。

```
CodeReviewAgent → ReviewResponseAgent → FixAgent
```

各エージェントは `.github/agents/*.agent.md` で定義されており、
`## 🔚 Post-Completion Required Steps` セクションに **完了後に自動呼び出すエージェント** を列挙している。

---

## 問題：各ステップが終端処理を持っていた

修正前、3つのエージェントそれぞれが post-completion に `@ArticleWriterAgent` と `@WorkSummaryAgent` を含んでいた。

```
# CodeReviewAgent（修正前）
Post-Completion:
  1. @ReviewResponseAgent
  2. @ArticleWriterAgent   ← ここに含まれていた
  3. @WorkSummaryAgent     ← ここに含まれていた

# ReviewResponseAgent（修正前）
Post-Completion:
  1. @FixAgent
  2. @ArticleWriterAgent   ← ここにも含まれていた
  3. @WorkSummaryAgent     ← ここにも含まれていた

# FixAgent（修正前・変更なし）
Post-Completion:
  1. @ArticleWriterAgent
  2. @WorkSummaryAgent
```

パイプラインが完走すると、連鎖の順に post-completion が実行されるため、
`@ArticleWriterAgent` と `@WorkSummaryAgent` がそれぞれ **3回ずつ** 呼び出されていた。

---

## 原因：「各ステップが終端を持つ」設計の落とし穴

エージェントチェーンを「安全側に倒す」ため、各エージェントに post-completion を持たせることがある。
しかしこの設計では **どのステップが起点になっても終端処理が実行される** という意図と、
**チェーン全体が完走したときに一度だけ実行される** という意図が混在してしまう。

今回の構造では：

- `CodeReviewAgent` → `ReviewResponseAgent` → `FixAgent` の順に動く
- それぞれが `@ArticleWriterAgent` を持つので、パイプライン完走時に3回呼ばれる
- 記事が3本、日記が3件、重複して生成される

---

## 修正：終端処理は末尾のエージェントだけに持たせる

「線形チェーンでは、終端処理を持つのは最後のエージェントだけ」という原則を適用した。

```
# CodeReviewAgent（修正後）
Post-Completion:
  1. @ReviewResponseAgent   ← 次のステップのみ

# ReviewResponseAgent（修正後）
Post-Completion:
  1. @FixAgent              ← 次のステップのみ

# FixAgent（変更なし・終端として機能）
Post-Completion:
  1. @ArticleWriterAgent
  2. @WorkSummaryAgent
```

パイプラインは次のような線形チェーンになり、`@ArticleWriterAgent` と `@WorkSummaryAgent` は
チェーン末尾で1回だけ呼ばれる。

```
CodeReview → ReviewResponse → Fix → ArticleWriter / WorkSummary
```

実際の diff は次のとおり。

```diff
# .github/agents/CodeReviewAgent.agent.md
-When all work is complete, you MUST call the following agents in order:
+When all work is complete, you MUST call the following agent:

 1. `@ReviewResponseAgent`
-2. `@ArticleWriterAgent`
-3. `@WorkSummaryAgent`
```

```diff
# .github/agents/ReviewResponseAgent.agent.md
-When all work is complete, you MUST call the following agents in order:
+When all work is complete, you MUST call the following agent:

 1. `@FixAgent`
-2. `@ArticleWriterAgent`
-3. `@WorkSummaryAgent`
```

---

## 設計パターンの比較

| パターン | 説明 | 向いているケース |
|---|---|---|
| **各ステップが終端処理を持つ** | どのステップが最後になっても確実に終端処理が動く | ステップが独立して呼び出されうる場合 |
| **終端のみが終端処理を持つ** | チェーン全体で一度だけ終端処理が動く | 常に線形チェーンとして動作する場合 |

今回のパイプラインは `CodeReviewAgent` が必ず起点となり `FixAgent` が必ず末尾となる線形構造のため、
後者のパターンが適切だった。

ただし、各エージェントを単独で呼び出す用途もある場合は、単独呼び出し時の挙動と
チェーン時の挙動を分けて定義するか、呼び出し側で制御する設計を検討する必要がある。

---

## まとめ

- マルチエージェントの post-completion は「そのエージェントが単独で完了したとき」に動く
- 線形チェーンで各ステップが同じ終端処理を持つと、チェーン完走時に重複実行される
- 線形チェーンでは **終端のエージェントだけ** に後処理（記事生成・日記保存など）を持たせるのがシンプルで正確
- 各エージェントを単独でも使う場合は、単独／チェーン兼用の設計を意識して定義する
