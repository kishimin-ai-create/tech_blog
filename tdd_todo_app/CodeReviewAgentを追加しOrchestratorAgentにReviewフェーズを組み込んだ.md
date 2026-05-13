# CodeReviewAgent を追加し OrchestratorAgent に Review フェーズを組み込んだ

## 対象読者

- GitHub Copilot のエージェント機能を使って開発フローを自動化したい開発者
- TDD を実践しており、レビュー工程もサイクルの一部として管理したい方
- AI エージェントの役割分担設計や命名ルール設計に興味がある方

---

## 背景：TDD サイクルにレビューが欠けていた

このプロジェクトでは OrchestratorAgent がマスターコンダクターとして TDD の各フェーズを制御している。従来のサイクルは次の 4 ステップだった。

```
Red → Green → Refactor → Integration & Report
```

Red Agent が失敗するテストを生成し、Green Agent が実装し、Refactor Agent がコード品質を改善する。しかしこの流れには **「完成したコードを誰が検証するか」** という問いへの答えがなかった。

Refactor が終わった後のコードがそのまま「コミット可能」として扱われていた。バグ、アーキテクチャ違反、テストの抜け漏れを機械的に検証する仕組みがなく、レビューは完全に人間の手作業に委ねられていた。

今回の作業はこの空白を埋めることを目的とした。

---

## 解決策：CodeReviewAgent の新規作成と OrchestratorAgent への統合

### 1. CodeReviewAgent の新規作成

`.github/agents/CodeReviewAgent.agent.md` を新たに作成した。このエージェントの役割は一つだ。

> コードの変更を読み、実際の問題点のみを `review/` に書き出す。

#### 出力フォーマット：優先度バッジ形式（P1 / P2 / P3）

指摘事項はすべて優先度バッジで分類して出力される。

| 優先度 | バッジ色 | 使用場面 |
| :----- | :------- | :------- |
| **P1** | orange   | バグ、セキュリティ脆弱性、壊れた CI、データ損失リスク |
| **P2** | yellow   | アーキテクチャ違反、エラーハンドリング不足、テストカバレッジ不足 |
| **P3** | blue     | 命名改善の提案、軽微なリファクタリング機会、ドキュメント不足 |

実際のレビューファイルのフォーマットは以下のようになる。

```markdown
## レビュー対象
<!-- 対象ブランチ・ファイル・コミット -->

## サマリー
<!-- 全体評価と主な懸念点を 3–5 行で -->

---

**![P1 Badge](https://img.shields.io/badge/P1-orange?style=flat) タイトル**

問題の説明。何が問題か、なぜ問題か、どうすれば直るかを具体的に。

Useful? React with 👍 / 👎.
```

P1 → P2 → P3 の順に並べることで、読み手が優先度の高い指摘から確認できるよう設計した。

#### ファイル名ルールの厳格化

出力先のファイル名は以下の形式に固定した。

```
review/{作業内容}-{YYYYMMDD}.md
```

`{作業内容}` の決定には優先順位がある。

1. ブランチ名からスラッグを生成（例：`feature/create-todo` → `create-todo`）
2. OrchestratorAgent 経由の場合は kebab-case スラッグを使用（例：`Todoリスト取得` → `todo-list`）
3. 変更ファイルの内容を要約した名前を使用（例：`CreateTodoInteractor修正`）

汎用名（`review`、`changes` など）の使用は **明示的に禁止** した。意味のないファイル名が蓄積すると、後から「何のレビューか」を把握するコストが跳ね上がるためだ。

OrchestratorAgent から呼び出す際は、エージェント間でスラッグを一致させる必要がある。OrchestratorAgent が `create-todo` というスラッグを指定したなら、CodeReviewAgent も `review/create-todo-YYYYMMDD.md` に書き出さなければならない。この制約を明記したことで、ファイルの存在確認チェックが機能するようになった。

日付の取得コマンドも OS 別に明記している。

```powershell
# Windows (PowerShell)
Get-Date -Format "yyyyMMdd"
```

```bash
# Unix
date "+%Y%m%d"
```

#### レビューしない項目の明確化

CodeReviewAgent は**フォーマット・インデント・空白に関する指摘を行わない**。これらはリンターやフォーマッターの仕事だ。エージェントが担うのは、ツールが検出できないロジック・アーキテクチャ・セキュリティ上の問題に限定した。

---

### 2. OrchestratorAgent の修正（v1.0.0 → v2.0.0）

`.github/agents/OrchestratorAgent.agent.md` に 2 種類の修正を加えた。

#### 修正①：壊れた PowerShell 残骸の除去

ファイル内に `$content = @'` のような PowerShell ヒアドキュメントの断片が混入していた。これは過去の編集ミスによるもので、YAML frontmatter として正しく解析されない状態になっていた。今回この残骸を除去し、frontmatter を正常な YAML に整形し直した。

#### 修正②：TDD サイクルへの Review フェーズ追加

サイクルを次のように拡張した。

```
変更前: Red → Green → Refactor → Integration & Report
変更後: Red → Green → Refactor → Review → Integration & Report
```

Phase 5 として Review Agent Execution が追加され、Refactor 完了後に必ず `@CodeReviewAgent` を呼び出すことが義務付けられた。

```
Phase 5: Review Agent Execution
- 変更されたファイルのリストを収集
- @CodeReviewAgent を呼び出し（変更ファイルのリストとフィーチャースペックを渡す）
- review/{feature-slug}-{YYYYMMDD}.md の存在を確認してから次のフェーズへ進む
```

**Definition of Done にレビューファイルの保存を追加した** ことも重要だ。

```markdown
- [ ] Red generates comprehensive test suite (all FAIL)
- [ ] Green generates implementation (all PASS)
- [ ] Refactor produces improved code (all PASS)
- [ ] CodeReviewAgent review file saved to review/{feature-slug}-YYYYMMDD.md  ← 追加
- [ ] File paths documented
- [ ] Status: ✅ Ready to Commit
```

レビューファイルが存在しない状態で "Ready to Commit" とマークすることが**禁止事項**として列挙された。これにより、レビューをスキップして完了扱いにするという抜け道がなくなった。

---

## 今回の変更がサイクルにどう影響するか

今回の変更後、OrchestratorAgent が一機能を実装する際の全体像は次のようになる。

```
1. 仕様ファイルを読み込む
2. Red Agent → 失敗するテストを生成
3. Green Agent → テストを通す実装を生成
4. Refactor Agent → コード品質を改善（テストはパスしたまま）
5. CodeReviewAgent → 変更全体をレビューし review/ にファイルを書き出す
6. レビューファイルの存在を確認
7. Integration & Report → ✅ Ready to Commit
```

従来は Step 5・6 が存在しなかった。Refactor が終わった時点で "完了" とみなされていたが、今後は **レビューファイルが存在しない限りコミット可能とマークできない**。

これはエージェントに「自己検証」の責任を持たせるという設計変更でもある。人間がレビューを忘れる、あるいはレビューのタイミングが遅れるという問題を、サイクルのルールそのものに組み込むことで防止する。

---

## 注意点

- **他のエージェントファイルへの波及確認**  
  OrchestratorAgent に PowerShell 残骸が混入していた原因は過去の編集ミスと思われる。他のエージェントファイルに同様の問題がないか確認しておくことを推奨する。

- **kebab-case スラッグの一貫性**  
  OrchestratorAgent と CodeReviewAgent の間でスラッグを一致させることは現在ルール上の要件だが、実際に一致しているかどうかの検証はエージェント自身の確認ロジックに依存している。命名変換の具体例を双方のファイルに揃えておくと混乱が減る。

- **P1/P2/P3 基準のドキュメント化**  
  現時点では各バッジの判断基準は `CodeReviewAgent.agent.md` 内の記述に依存している。将来的に `docs/` 配下に独立したドキュメントとして切り出すと、他エージェントが参照しやすくなる。

---

## まとめ

今回の作業で TDD サイクルに **Review フェーズ** が正式に組み込まれた。変更の要点を整理すると次のとおりだ。

| 変更 | 内容 |
| :--- | :--- |
| `CodeReviewAgent` 新規作成 | P1/P2/P3 バッジ形式でレビュー結果を `review/` に書き出す |
| `OrchestratorAgent` v2.0.0 | TDD サイクルに Review フェーズを追加、レビューファイル保存を完了条件に設定 |

エージェントがコードを生成するだけでなく、**自分たちが生成したコードを自分たちでレビューする**という仕組みが整った。これにより、レビューが遅れる・スキップされるという人間側の問題を、サイクルのルールそのものに組み込んで防止できるようになった。
