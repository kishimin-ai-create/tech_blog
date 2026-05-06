# FixAgentのバグ修正にRed→Green→Refactorサイクルを義務化した

## 対象読者

- TDD に興味があるエンジニア
- AI エージェントにバグ修正を任せている、または検討しているチーム
- FixAgent のルールファイルがどのように設計されているか知りたい人

## この記事でわかること

`.github/agents/FixAgent.agent.md` に「Bug Fix TDD Cycle」セクションを追加し、バグ修正を Red → Green → Refactor の順序で行うことを義務化した背景と変更内容を解説します。コミット `84f39e0` の変更です。

---

## 背景：FixAgent はどんなエージェントか

FixAgent は「壊れているものを正しく直す」ことに特化した修正専門エージェントです。その設計哲学は **"surgical repair specialist"**——最小限の変更でバグを直し、関係のないコードには一切触らないというものです。

従来の FixAgent には次のような作業ステップが定義されていました。

1. 根本原因の特定
2. 最小変更の適用
3. typecheck + lint + テストで検証
4. コミット

これ自体は合理的ですが、「どのタイミングでテストを書くか」が明示されていませんでした。

---

## 問題：「テストを書く」タイミングが曖昧だった

テストを書くタイミングが未定義のままだと、実際のバグ修正フローで次のような問題が起きえます。

- **先にコードを直してからテストを書く**：修正後にテストを書くと「通過することを確認してからテストを追加する」だけになりがちで、そのテストがバグを本当に再現できているか保証されない
- **バグが再現しないテストを書いてしまう**：テストが最初から GREEN だと、バグの根本原因を取り違えていても気づかない
- **修正とテストを別コミットにする**：後からコードだけ見たとき、「このテストはどのバグに対応したものか」が追跡しにくくなる

---

## 変更内容：Bug Fix TDD Cycle の追加

FixAgent に以下の7ステップからなる必須サイクルを追加しました。

```
1. 🔴 RED     — バグを再現する失敗テストを書く
2. ✅ VERIFY  — テストを実行し、失敗することを確認する（バグの存在証明）
3. 🟢 GREEN   — テストを通す最小限のコード変更を行う
4. ✅ VERIFY  — テストスイート全体を実行し、すべて通ることを確認する
5. 🔵 REFACTOR（任意） — 触ったコードだけを整理する。挙動は変えない
6. ✅ VERIFY  — リファクタリング後にテストが壊れていないことを確認する
7. 💾 COMMIT  — テストと修正を1コミットにまとめる
```

### 各ステップのルール

**Step 1（失敗テストを書く）**
テストは報告された症状を直接再現するものでなければなりません。コンパイルエラーや import エラーで失敗するテストは「バグの再現」ではないため、対象の層に適切なテストを配置します。

**Step 2（RED の確認）**
新しいテストだけを先に実行します。コード修正前にテストが通ってしまう場合は、根本原因の認識が誤っている可能性があるため、立ち止まって再調査します。

**Step 3（最小変更）**
失敗しているテストが通る最小限の変更のみ行います。テストがカバーしていない箇所は修正しません。

**Step 4（GREEN の確認）**
新しいテストだけでなく、テストスイート全体を実行します。リグレッションが発生していないことの確認です。

**Step 5（リファクタリング）**
任意ステップです。修正によって生じた明らかな重複や不明確な命名がある場合のみ実施します。**修正スコープ外のコードは絶対にリファクタリングしません。**

**Step 7（コミット）**
テストと修正を1つのコミットにまとめます。コミットメッセージには根本原因を記載します。

---

## Verification Commands の追加

`## ✅ Mandatory Verification Commands` セクションに、各ステップ対応のコマンドを追加しました。

```bash
# Step 1 — 新しいテストだけを実行（修正前：失敗することを確認）
npm run test -- --reporter=verbose <path-to-new-test>

# Step 2 — 修正を適用後、新しいテストだけを実行（通ることを確認）
npm run test -- --reporter=verbose <path-to-new-test>

# Step 3 — 全スイートを実行（リグレッションがないことを確認）
npm run typecheck   # 0 errors
npm run lint        # 0 errors
npm run test        # all pass
```

---

## Definition of Done の更新

「完了の定義」チェックリストに2項目を追加しました。

```markdown
- [ ] 修正前に失敗テストを書いた（🔴 RED 確認）
- [ ] その失敗テストが修正後に通る（🟢 GREEN 確認）
```

また、コミット粒度のルールも更新しました。

```markdown
# 修正前
- [ ] Each fix committed individually after verification

# 修正後
- [ ] Test and fix committed together in one commit per defect
```

テストと修正が常に同じコミットに存在することで、git blame や git bisect でのトレーサビリティが向上します。

---

## Pre-Fix / Post-Fix チェックリストの更新

**Pre-Fix チェックリスト**（修正着手前）に追加:

```markdown
- [ ] A failing test that reproduces the bug has been written
- [ ] The failing test has been run and confirmed to fail (🔴 RED)
```

**Post-Fix チェックリスト**（修正後）に追加:

```markdown
- [ ] The newly written test now passes (🟢 GREEN confirmed)
- [ ] Refactoring (if any) is scoped only to code touched by the fix
- [ ] Verified and committed (test + fix in one commit)  ← 「一緒にコミット」を明記
```

---

## なぜ AI エージェントに TDD サイクルを課すか

通常の開発者であれば「テストを書く→失敗を確認→直す」という流れは経験から身につきますが、AI エージェントはルールファイルに書かれていないことは実行しません。

- ルールがなければ「直ってそうだからテストを後から書く」という順序になる可能性がある
- RED を確認せずに GREEN テストを追加しても「バグが再現できるテスト」の保証にならない
- 修正とテストが別コミットになると、後から変更の意図が追えなくなる

ルールファイルへの追記は「エージェントに正しい習慣を強制するインターフェース」として機能します。

---

## まとめ

| 変更箇所 | 変更内容 |
|---|---|
| 新セクション `Bug Fix TDD Cycle` | 7ステップの必須サイクルを定義 |
| `Mandatory Verification Commands` | Step別のコマンドを追加 |
| `Pre-Fix Checklist` | 失敗テスト作成・RED 確認を追加 |
| `Post-Fix Checklist` | GREEN 確認・リファクタリングスコープ制限を追加 |
| `Definition of Done` | RED/GREEN 確認チェックボックスを追加 |
| コミット粒度ルール | テストと修正を1コミットに統一 |

TDD の Red→Green→Refactor は「テスト駆動開発の作法」として広く知られていますが、AI エージェントベースの開発フローでも同様に有効です。ルールファイルに明文化することで、エージェントが自律的にバグ修正する際の品質基準を担保できます。
