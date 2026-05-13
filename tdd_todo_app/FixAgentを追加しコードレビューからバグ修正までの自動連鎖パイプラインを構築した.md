# FixAgent を追加しコードレビューからバグ修正までの自動連鎖パイプラインを構築した

## 背景

このリポジトリでは、コードの品質向上や修正を AI エージェントに委任する仕組みを整備してきた。これまでは「品質向上」と「バグ修正」の両方を `@RefactorAgent` が担当していたが、役割が混濁していた。

- **リファクタリング**は「外部動作を変えずに内部構造を改善する」作業だ
- **バグ修正**は「間違っている動作を正しくする」作業であり、必然的に外部動作が変わる

この2つを同じエージェントに担わせると、エージェントが「テストを壊してはいけない」という RefactorAgent の制約と「間違ったテストを直してよい」というバグ修正の許可が衝突する。また、レビューで指摘された問題を自動的に修正する連鎖パイプラインを作るにも、専門のエージェントが必要だった。

---

## FixAgent の設計

### RefactorAgent との役割の違い

| 観点 | RefactorAgent | FixAgent |
|---|---|---|
| 目的 | 内部構造の改善 | 誤った動作の修正 |
| 外部動作の変更 | ❌ 禁止 | ✅ 必要なら変更する |
| テストの修正 | ❌ テストは不変の真実 | ✅ テスト自体が間違っている場合は修正可 |
| commit prefix | `refactor:` | `fix:` |
| `user-invocable` | `false`（他エージェントが呼ぶ） | `true`（直接呼び出し可） |
| 哲学 | 「安全なリファクタリング職人」 | 「外科的修復の専門家」 |

### FixAgent が許可される変更の例

```
✅ 誤ったロジックが間違った出力を生む場合 → 修正する
✅ null/undefined ガードが欠けていてランタイムエラーが起きる → ガードを追加する
✅ 間違ったステータスコードやエラーコード → 正しい値に直す
✅ コンパイルエラーを引き起こす型アノテーションの誤り → 修正する
✅ 仕様書と異なるビジネスルールの実装 → 仕様に合わせる
```

### 1 fix per commit ルール

RefactorAgent と同様、FixAgent も **1つの修正ごとに個別にコミットする**。複数の問題が見つかった場合でも、修正 → 検証（`typecheck + lint + test`） → コミット を繰り返す。これにより、どのコミットでどの問題が修正されたかが Git の履歴から追跡できる。

```
fix: add null guard in TodoRepository.findById
fix: correct HTTP 500 to 409 on duplicate app name
fix: fix incorrect deletedAt comparison in soft-delete query
```

---

## エージェント連鎖パイプラインの設計

FixAgent を追加したことで、コードレビューから修正完了まで自動的につながるパイプラインを構築できた。

### 連鎖の全体像

```
@CodeReviewAgent
    ↓ レビューファイルを review/ に出力
@ReviewResponseAgent
    ↓ 指摘への返答 + 修正が必要な指摘を特定
@FixAgent
    ↓ コードの修正・検証・コミット
@ArticleWriterAgent + @WorkSummaryAgent
    ↓ 技術記事と作業日記を出力
```

各エージェントの `Post-Completion Required Steps` に次に呼ぶエージェントを定義することで、この連鎖を実現した。

### CodeReviewAgent の post-completion 設定

```markdown
## 🔚 Post-Completion Required Steps

When all work is complete, you MUST call the following agents in order:

1. `@ReviewResponseAgent` — Address each finding in the review file
2. `@ArticleWriterAgent` — Save changes as a technical article under `blog/`
3. `@WorkSummaryAgent` — Save work as a diary entry to `diary/YYYYMMDD.md`
```

### ReviewResponseAgent の post-completion 設定

```markdown
## 🔚 Post-Completion Required Steps

When all work is complete, you MUST call the following agents in order:

1. `@FixAgent` — Request implementation of any review findings that require code fixes
2. `@ArticleWriterAgent` — Save changes as a technical article under `blog/`
3. `@WorkSummaryAgent` — Save work as a diary entry to `diary/YYYYMMDD.md`
```

`@CodeReviewAgent` を一度呼ぶだけで、レビュー → 返答 → 修正 → 記事・日記 という一連のフローが自動で流れる。

---

## 設計で考慮した点

### RefactorAgent を呼び出し不可にする理由

`RefactorAgent` は `user-invocable: false` のままにしてある。これは、RefactorAgent が OrchestratorAgent の Red → Green → **Refactor** サイクルの中でのみ使われるべきエージェントだからだ。ユーザーが直接 `@RefactorAgent` と書いてリファクタリングを依頼した場合、OrchestratorAgent が管理する TDD サイクル外で動作してしまい、テストとの整合性が保証されない可能性がある。

### FixAgent はテスト修正を「許可するが慎重に」

FixAgent は「テスト自体が間違っている場合は修正可」としているが、仕様書による確認を条件としている。仕様書のない状況でテストを修正することは許可されない。これにより「テストを壊してバグを隠す」という誤った使われ方を防ぐ。

### 連鎖における Article/WorkSummary の重複

各エージェントが `@ArticleWriterAgent` と `@WorkSummaryAgent` を呼ぶため、パイプラインが全て流れると複数の記事と日記エントリが生成される。これは意図的な設計だ。各エージェントが**自分の作業内容に特化した**記事を書く方が、全体をまとめた1本の記事よりも検索性・再利用性が高い。

---

## まとめ

今回の変更のポイント:

1. **役割の分離**: リファクタリング（外部動作を変えない）とバグ修正（外部動作を正す）を別エージェントに担わせ、それぞれの制約を明確にした
2. **自動連鎖**: `Post-Completion Required Steps` でエージェントを数珠つなぎにし、`@CodeReviewAgent` を呼ぶだけでレビュー→修正→記録が自動で流れるようにした
3. **commit per fix**: FixAgent も RefactorAgent と同じく1修正ごとにコミットし、変更履歴を細かく残す

これにより、コードレビューから修正完了まで人間の介入を最小限に抑えながら、各ステップの成果物（レビューファイル・修正コミット・技術記事・作業日記）が自動で生成されるパイプラインが完成した。
