# GitHubのPRテンプレート認識をファイル名修正でそろえた

## 対象読者

- GitHub の PR テンプレートをリポジトリ運用に組み込んでいる人
- レビュー指摘をもとに設定やドキュメントの整合性を直したい人

## この記事の範囲

この記事では、レビュー対応として行った **PR テンプレートのファイル名修正** と、それに伴う **関連参照の更新** を扱います。  
`PullRequestWriterAgent` 自体の全体設計や、GitHub 上での実動確認までは扱いません。

## 背景

今回のレビューでは、PR テンプレートのファイル名が `.github/pull-request_template.md` になっている点が指摘されました。  
`review/PR下書き生成用のPullRequestWriterAgentを追加-20260423.md` では、GitHub が PR 本文の自動展開に使う認識済みパスとして `.github/pull_request_template.md` のような **アンダースコア区切り** を前提にしていることが示されています。

このため、ファイル名が `pull-request_template.md` のままだと、GitHub 上で PR を開いてもテンプレートの各セクションが自動で入らず、意図した PR フォーマットを通しにくい状態でした。

## なぜアンダースコアのファイル名が重要だったのか

問題の中心は、テンプレートの中身ではなく **GitHub が認識するファイル名規約** でした。

- 問題のあった名前: `.github/pull-request_template.md`
- 修正後の名前: `.github/pull_request_template.md`

レビュー指摘の内容に基づくと、GitHub は後者のような認識済みファイル名でないと PR 本文の自動入力に使いません。  
つまり、テンプレート本文を整備していても、**ファイル名が規約に合っていなければ運用に乗らない**、というのが今回の修正理由です。

## 変更したこと

### 1. PR テンプレートをリネームした

次の修正を行いました。

- `.github/pull-request_template.md`
- ↓
- `.github/pull_request_template.md`

これにより、リポジトリ内で使っている PR テンプレートの配置を、レビュー指摘のあった GitHub 認識前提のファイル名に合わせました。

### 2. 古いテンプレートパスに結びついた参照を更新した

テンプレート名だけを直すと、関連ドキュメントやエージェント定義に古い参照が残ります。  
そのため、次のファイルでもテンプレート参照を `.github/pull_request_template.md` にそろえました。

- `.github/agents/PullRequestWriterAgent.agent.md`
- `diary/20260423.md`
- `blog/実装内容を反映したPR下書きを作るPullRequestWriterAgentを整備した.md`
- `pull-request/PR下書き生成用のPullRequestWriterAgentを追加.md`

特に `.github/agents/PullRequestWriterAgent.agent.md` では、リポジトリの PR テンプレートとして `.github/pull_request_template.md` を参照する前提が明記されています。  
関連する日誌、ブログ記事、PR 下書きファイルも同じ参照先にそろえたことで、テンプレート運用の説明と実ファイルの対応が一致しました。

## レビュー対応の中で、返信のみで済んだ項目

今回のレビューでは、すべてが修正対象だったわけではありません。  
次の 2 件は、確認時点でリポジトリがすでに指摘内容を満たしていたため、**返信のみ** の対応になっています。

- `backend/tsconfig.json` には、すでに ES2022 の `target` / `lib` 関連設定が入っていた
- `.github/workflows/playwright.yml` は、すでにリポジトリ直下の `workflows` 配下にあり、`frontend` の `working-directory` も設定されていた

実際に `backend/tsconfig.json` では `target: "ES2022"` と `lib: ["ES2022", "DOM", "DOM.Iterable"]` が確認できます。  
また `.github/workflows/playwright.yml` でも、ルートのワークフロー配置と `frontend` を作業ディレクトリにする設定が入っています。

## 実装上のポイント

今回の変更は大きな機能追加ではなく、**命名規約と参照整合性の修正** です。  
ただし、こうした修正は運用面では重要です。

- GitHub が認識するファイル名に合わせる
- 変更後のパスを関連ドキュメントにも反映する
- すでに適合済みの指摘は、不要な変更を加えず返信で処理する

レビュー対応では、修正そのものだけでなく、**どこまでが修正対象で、どこからが既存状態の確認で済むか** を切り分けることが大切だと分かります。

## まとめ

今回のレビュー対応では、PR テンプレートを GitHub が認識するファイル名に合わせて、

- `.github/pull-request_template.md` を `.github/pull_request_template.md` に変更し
- 関連する 4 ファイルのテンプレート参照を更新し
- すでに満たしていた `backend/tsconfig.json` と `.github/workflows/playwright.yml` の指摘は返信のみで処理しました

ポイントは、**テンプレートの内容ではなく、GitHub に認識されるファイル名であること** が運用上の前提だった点です。  
レビュー起点の修正として、ファイル名と参照先の整合性をそろえた変更でした。
