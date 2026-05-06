# 削除済みファイルへのコードレビュー指摘（陳腐化した finding）にどう返答するか

## 対象読者

- チームやエージェントが生成したコードレビューを運用しているエンジニア
- レビュー指摘の返答（ReviewResponse）フローを整備しているチーム
- 過去のコードベースに向けた「的外れな指摘」を受けたことがある開発者

---

## 問題の背景

コードレビューは「現在のコードベース」に基づいて行われるべきだが、実際の運用では**タイムラグ**が生じることがある。特に以下のケースで発生しやすい。

- レビューがキューに積まれている間に別のセッションがコードを変更した
- 自動レビューエージェントが古いスナップショットに対してコメントを生成した
- PR のマージ後に別ブランチで削除・移動されたファイルへの指摘が残っている

今回のレビューでは、OpenAPI の server URL に `/api/v1` が二重になっているという指摘（P2）が **2 件**届いた。

```
The route metadata already registers paths with the `/api/v1` prefix,
so setting servers to `/api/v1` makes clients call `/api/v1/api/v1/...`
```

これは正当な指摘内容ではあった。しかし **問題の YAML ファイルは前のセッションですでに削除済み**だった。

```bash
git log --oneline | grep "delete.*OpenAPI"
# 8d21575 delete: remove OpenAPI specification for TDD Todo App API
```

対象ファイルがリポジトリに存在しない以上、修正するものがない。

---

## 陳腐化（stale）な finding の判定基準

finding が「陳腐化している」と判断できる条件は次のとおり。

| 条件 | 説明 |
|---|---|
| 対象ファイルが削除されている | `git log` や `ls` で存在を確認できない |
| 対象コードがすでに別の方法で修正済み | 同じ問題が別コミットで直っている |
| 指摘箇所が別ファイルに移動・リネームされた | ファイル名やパスが変わっている |

「まだ該当コードがある気がする」という思い込みは危険。必ず**現在のリポジトリの状態**を確認してから判定する。

---

## 返答の書き方

陳腐化した finding への返答は、以下の3点を簡潔に伝えれば十分だ。

1. **対応区分を明示する**（`reply-only` / `fixed` / `wont-fix` など）
2. **なぜ対応不要かを説明する**（ファイルが削除済み、コードが変更済みなど）
3. **コミット SHA やファイル名など根拠を添える**

今回の返答例：

```
Disposition: reply-only — stale finding

The OpenAPI YAML file that contained the `/api/v1` server URL was deleted
in a prior session. No `.yaml` file exists under `backend/` anymore,
so there is nothing to correct. This finding no longer applies to the
current codebase.
```

「やることがない」という判断を明確に記録しておくことが重要だ。放置すると後から「なぜこの指摘が未対応なのか」と疑問を持たれる。

---

## 修正すべき finding との違い

今回のレビューには陳腐化した P2 が2件、実際に修正すべき P1 が1件含まれていた。

| Finding | 優先度 | 内容 | 対応 |
|---|---|---|---|
| OpenAPI server URL の二重 /api/v1 | P2 | YAML ファイル削除済み | reply-only（陳腐化） |
| npm run dev が .env を読み込まない | P1 | `dev` スクリプトに `--env-file-if-exists` がない | コード修正 |
| OpenAPI server URL の二重 /api/v1 (重複) | P2 | Finding 1 と同内容、YAML 削除済み | reply-only（陳腐化） |

陳腐化した指摘を「対応済みとして処理してしまう」か「完全にスルーする」かという誤りを避けるため、**reply-only として明示的に記録する**運用が適切だ。

---

## 自動レビューパイプラインでの注意

このリポジトリでは `CodeReviewAgent → ReviewResponseAgent → FixAgent` という自動パイプラインが動いている。自動化環境では陳腐化の判定が特に重要になる。

- **レビュー生成タイミングと修正タイミングのずれ**が起きやすい
- エージェントが「対象が存在しない finding」を自動修正しようとすると空振りコミットや誤修正が生まれる
- FixAgent が実行される前に ReviewResponseAgent が現在のリポジトリ状態を確認し、「修正不要」と判定する責務を持つことでパイプライン全体の品質が保たれる

---

## まとめ

- レビュー指摘は「現在のコードベース」に対して有効かどうかを必ず確認する
- 対象ファイルが削除済みの場合は `reply-only — stale finding` として明示的に返答し、変更なしを記録する
- 陳腐化した finding を黙ってスキップするのではなく、**なぜ対応しないのかを文書化**することが後のレビュアーへの説明責任になる
- 自動レビューパイプラインでは、陳腐化の判定を FixAgent 実行前に行うステップが品質の要になる
