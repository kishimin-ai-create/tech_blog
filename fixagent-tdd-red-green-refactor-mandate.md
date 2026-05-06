# FixAgentのバグ修正にRed→Green→Refactorサイクルを義務化した

## 対象読者

- AI エージェントにバグ修正を任せているチームで、修正の質やプロセスのばらつきが気になっている人
- TDD のサイクルを開発ルールとして明文化したい人
- エージェント定義ファイル（`.agent.md`）の設計に興味がある人

---

## 背景

このプロジェクトでは `FixAgent` という GitHub Copilot 向けエージェントがバグ修正を担当する。修正の手順はもともと「根本原因を特定する → 最小限の変更を加える → typecheck + lint + test で検証する → コミット」として定義されていた。

しかしフロントエンド CI 整備の PR に対するコードレビューで実際に5件のバグを修正したとき、**テストが先に存在せず、修正後にテストを後付けするパターン**に気づいた。修正は正しかったが、TDD の「先にテストを書く」という手順が守られていなかった。

この問題を解消するために、`FixAgent.agent.md` に **Bug Fix TDD Cycle（必須）** セクションを追加した。

---

## 変更内容

### 追加されたTDDサイクル定義

```
1. 🔴 RED   — バグを再現する失敗テストを書く
2. ✅ VERIFY — テストを実行し、失敗することを確認する（バグが存在することの証明）
3. 🟢 GREEN  — テストをパスする最小限のコード変更を書く
4. ✅ VERIFY — テストスイート全体を実行し、全テストがパスすることを確認する
5. 🔵 REFACTOR（任意） — 変更したコードだけをクリーンアップする（挙動は変えない）
6. ✅ VERIFY — リファクタリング後もテストがパスすることを確認する
7. 💾 COMMIT — テストと修正をひとつのコミットにまとめてコミットする
```

各ステップの説明だけでなく、**スキップが禁止されている**ことも明記された。

---

## なぜこの変更が必要だったか

### テストを後付けすることの問題

修正を先に書いてからテストを追加するパターンには、見えにくいリスクがある。

**テストがバグを正確に再現しているかどうかを確認できない**。テストを修正後に書くと、そのテストは現在の（修正済みの）実装に合わせて書かれやすい。つまり、そのテストが「バグが存在したときに本当に失敗したか」を誰も確認していない。

RED ステップを省略すると、意味のないテストが生まれる可能性がある。テストは緑になるが、バグが再発しても緑のままという状態だ。

### TDD サイクルの各ステップが持つ意味

| ステップ | 役割 |
|----------|------|
| 🔴 RED | 「バグが実在する」ことをテストで証明する |
| ✅ VERIFY (red) | テストがバグを正確に再現していることを確認する |
| 🟢 GREEN | テストをパスするための最小変更を書く |
| ✅ VERIFY (green) | 修正がリグレッションを生んでいないことを確認する |
| 🔵 REFACTOR | 修正で生じたコードの汚れを除去する（任意） |
| 💾 COMMIT | テストと修正を同一コミットに含める |

重要なのはステップ2（`VERIFY RED`）だ。テストを実行して失敗することを確認しなければ、「そのテストは修正前のコードで落ちる」という前提が証明されない。テストを書いただけでは不十分で、実際に実行して RED であることを見届ける必要がある。

### エージェントにルールを課す意味

AI エージェントは「正しい出力を出す」ことには優れているが、**手順を守る**ことは明示的なルールがないとぶれやすい。

エージェント定義ファイル (`.agent.md`) は、そのエージェントが何をすべきかだけでなく「どのように進めるか」を規定するドキュメントだ。TDD サイクルをエージェント定義に組み込むことで、Copilot がバグ修正タスクを処理するたびにこの手順に従うよう制約できる。

---

## FixAgent の定義ファイルにおける位置づけ

`FixAgent.agent.md` は次のセクションで構成されている。

1. **Role** — FixAgent が何をするエージェントか
2. **Bug Fix TDD Cycle（今回追加）** — 全バグ修正で必須のサイクル
3. **Input** — 受け取る情報の種類
4. **Output** — 成果物の形式
5. **Strict Rules** — 変えてはいけない制約
6. **Thinking Rules** — 思考の進め方
7. **Definition of Done** — 完了の定義

TDD サイクルを **Role の直後に配置した**のは意図的だ。FixAgent のアイデンティティ（「外科的修正の専門家」）を説明した直後に「この手順で必ずやること」を提示することで、サイクルが「オプションの参考情報」ではなく「この役割の核心」として位置づけられる。

---

## 定義ファイルの一部

```markdown
## 🔴 Bug Fix TDD Cycle (Mandatory for all bug fixes)

Every bug fix **MUST** follow this cycle, in order. Do not skip steps.

1. 🔴 RED   — Write a failing test that reproduces the bug
2. ✅ VERIFY — Run the test and confirm it fails (proves the bug exists)
3. 🟢 GREEN  — Write the minimal code change that makes the test pass
4. ✅ VERIFY — Run the test suite and confirm all tests pass
5. 🔵 REFACTOR (optional) — Clean up only the code you just touched
6. ✅ VERIFY — Re-run tests to confirm refactoring did not break anything
7. 💾 COMMIT — Commit once, with both the test and the fix together
```

「Do not skip steps」という一文を明示したのは、AI エージェントが「効率化のためにステップを飛ばす」という判断をしにくくするためだ。

---

## 完了の定義（Definition of Done）の更新

TDD サイクルの追加に合わせて、FixAgent の完了条件も更新された。

```markdown
## ✅ Definition of Done

- [ ] Root cause identified and documented in the commit message
- [ ] A failing test reproducing the bug was written before the fix (🔴 RED confirmed)
- [ ] The fix makes that test pass (🟢 GREEN confirmed)
- [ ] All tests pass — confirmed by running `npm run test`
- ...
```

「🔴 RED confirmed」「🟢 GREEN confirmed」をチェックボックスとして明示することで、エージェントが自己評価する際に「テストが先に失敗したことを確認したか」が問われる形になっている。

---

## まとめ

- FixAgent のバグ修正プロセスに **Red → Green → Refactor の TDD サイクルを必須手順として追加**した
- 特に重要なのは **RED の確認ステップ**で、修正前にテストが失敗することを実際に実行して確認することでテストの有効性を保証する
- エージェント定義ファイルへの組み込みは「AI が手順を守る」ための最も直接的な方法だ — 言葉で「TDD でやってください」と毎回指示するのではなく、エージェントの動作仕様として書き込む
- テストと修正を同一コミットに含める規則により、将来の git bisect やリグレッション調査がしやすくなる
