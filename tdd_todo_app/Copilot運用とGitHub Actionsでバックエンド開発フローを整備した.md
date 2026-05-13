# Copilot 運用と GitHub Actions でバックエンド開発フローを整備した

## 対象読者

- GitHub Copilot CLI を使ったマルチエージェント開発運用に興味がある人
- TDD・レビュー・PR 下書き・記事作成を一連のフローにまとめたい開発者
- `copilot-setup-steps.yml` のような環境整備 workflow を設計している人

## この記事で扱う範囲

- `copilot-setup-steps.yml` による Copilot 実行環境の再現性確保
- `ArticleWriterAgent` / `WorkSummaryAgent` による変更記録の運用
- `@AgentName` 呼び出しを優先する `CUSTOM_COMMANDS.md` の設計
- TDD フローの Review フェーズ統合（CodeReviewAgent・OrchestratorAgent）
- `CodeReviewAgent` と `PullRequestWriterAgent` による後工程の自動化

> 注: この記事は `git` コマンドを直接実行できない状況で書かれており、観測可能なリポジトリファイルをもとに整理しています。

---

## Copilot 用セットアップ workflow が運用の再現性を上げている

`.github/workflows/copilot-setup-steps.yml` では、Copilot CLI がリポジトリ文脈で動く前提条件を次のように自動化している。

- checkout
- Node.js 22 セットアップ
- `backend/package-lock.json` と `frontend/package-lock.json` を使った npm cache
- `backend/` で `npm ci`
- `frontend/` で `npm ci`
- `git --version`
- `pwsh --version`

これは単なる環境構築メモではなく、**Copilot CLI がリポジトリ固有の依存関係を前提に動く状態を固定する**ためのものだ。backend と frontend の両方を同じ workflow で初期化しているのは、マルチパッケージ構成での設定ずれを減らす。

---

## 「変更を残す仕組み」が強くなっている

コードを書いて終わりにしない—この運用方針が、エージェント定義に明示的に組み込まれている。

### `ArticleWriterAgent`：実装を blog に残す前提が明文化された

`.github/agents/ArticleWriterAgent.agent.md` では次のルールが定義されている。

- git evidence を優先する
- 仕様・diff・PR 文脈を読む
- 日本語の技術記事にする
- `blog/` に Markdown で保存する
- rules が設計を形作ったなら、その関係を説明する

つまり「コードを書いたら終わり」ではなく、**変更理由と設計意図まで記録に残す**運用が agent に埋め込まれている。

### `WorkSummaryAgent`：要約を chat で終わらせず `diary/` に追記する

`.github/agents/WorkSummaryAgent.agent.md` の設計は実務寄りだ。

- `diary/YYYYMMDD.md` に書く
- 同日ファイルには append する
- incomplete work も書く
- git diff が取れなければそのことを明記する

実際に `diary/20260425.md` には、OrchestratorAgent のルール強化・Todo API 実装・TypeScript rules 追加・WorkSummaryAgent 強化・Copilot setup 整備がまとまっており、**後から repository を読む人間にとっての時系列ログ**になっている。

### `CUSTOM_COMMANDS.md`：prompt より `@AgentName` を優先する方針

`.github/CUSTOM_COMMANDS.md` には次の役割が整理されている。

| コマンド | 役割 |
|---|---|
| `@ArticleWriterAgent` | 技術記事を `blog/` に出力する |
| `@OpenApiWriterAgent` | backend 実装をもとに `docs/spec/backend/openapi.yaml` を更新する |
| `@WorkSummaryAgent` | 作業内容を日本語で要約し `diary/` に追記する |
| `@ReviewResponseAgent` | `review/` を入力に修正方針と返信文をまとめる |

特に重要なのは、Copilot CLI では prompt file が必ずしも slash command として安定しないため、**実運用では `@AgentName` を優先する**と明言している点だ。これはツールの理想論ではなく、CLI の現実に合わせた設計だ。

---

## TDD フロー自体も、レビュー込みで閉じるように強化されている

`.github/agents/OrchestratorAgent.agent.md` を見ると、TDD の定義が変わっている。

以前の 3 段階ではなく、現在は次の 5 段階だ。

```
1. Red
2. Green
3. Refactor
4. Review
5. Integration & Report
```

しかも `OrchestratorAgent` は、自分でコードを書いてはいけないと明示されている。

- sub-agent が失敗しても自分で source を書かない
- 1 回だけ retry し、それでもだめなら止まる
- review file ができる前に完了扱いしない

`diary/20260425.md` にも、このルール強化の背景として「オーケストレーターが失敗を埋めるために自分でコードを書いてしまった」問題が記録されている。

自動化を進めると「とにかく結果を出す」方向へ寄りがちだが、このリポジトリでは逆に、**役割の境界を壊さないことを優先**している。

---

## CodeReviewAgent と PullRequestWriterAgent まで後工程がつながっている

### `CodeReviewAgent`

`.github/agents/CodeReviewAgent.agent.md` は、レビューを `review/{topic}-{YYYYMMDD}.md` に保存する前提で作られている。

review checklist には次が並んでいる。

- Clean Architecture 違反
- handler/controller の薄さ
- infrastructure error のマッピング
- test quality
- security

実際の `review/todo-api-20260425.md` でも、この観点から `Boolean()` coercion・cascade delete テストの検証不足・trim と一意制約の不一致・timestamp の一貫性が指摘されていた。

### `PullRequestWriterAgent`

`.github/agents/PullRequestWriterAgent.agent.md` では、PR 説明文まで repository 内で標準化している。固定セクションは次だ。

- `## Summary`
- `## Related Tasks`
- `## What was done`
- `## What is not included`
- `## Impact`
- `## Testing`
- `## Notes`

出力先は `pull-request/` 配下の Markdown で、ファイル名は PR タイトルから導かれる。

---

## この構成が示すこと

ここまで来ると、このリポジトリは単にアプリを作る場所というより、**実装 → テスト → レビュー → PR → 記事 → 日報**を一連の作業として設計していると言える。

各工程を担う agent が定義されており、共通前提は `instructions` / `rules` に寄せられている。Copilot CLI の実運用に合わせて `@AgentName` 優先の呼び出し方にまとめた `CUSTOM_COMMANDS.md` がそのインデックスになっている。

---

## まとめ

今回整備された Copilot 運用と GitHub Actions のポイントは次のとおりだ。

- `copilot-setup-steps.yml` で Copilot 実行環境の前提条件を固定した
- `ArticleWriterAgent` / `WorkSummaryAgent` で変更内容を `blog/` と `diary/` に残す運用を agent に組み込んだ
- `CUSTOM_COMMANDS.md` に `@AgentName` 優先の呼び出し方をまとめた
- OrchestratorAgent に Review フェーズを追加し、レビューファイルなしの完了を禁止した
- `CodeReviewAgent` と `PullRequestWriterAgent` で後工程をリポジトリ内で完結させた

設計ルールが「書くだけ」で終わらず、**workflow・agent・diary・blog まで一貫して落とし込まれている**ことが、この開発フロー整備の核心だ。
