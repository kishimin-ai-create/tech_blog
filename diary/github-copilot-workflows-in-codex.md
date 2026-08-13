# GitHub Copilot 向けの共有エージェント運用を Codex でも使えるようにした話

## はじめに

Codex でも再利用できるようにするための橋渡しでした。

## 対象読者

- `.github/agents` や `.github/prompts` を使って AI 向けの運用を整理している人
- GitHub Copilot と Codex の両方で同じリポジトリルールを使いたい人
- リポジトリ内の指示書を source of truth のまま再利用したい人

## この記事のスコープ

この記事で扱うのは、`a39bafd` から `803ad82` までのコミットで入った
Codex 対応です。具体的には次のファイル群を対象にしています。

- `.codex/agents/`
- `.agents/skills/`
- `.github/CODEX_AGENT_BRIDGE.md`
- `AGENTS.md`
- `.codex/README.md`

一方で、GitHub 上でしか成立しない機能や Actions のような話は扱いません。

## 背景

このリポジトリには、もともと GitHub Copilot 向けの共有資産がありました。

- `.github/agents/*.agent.md`
- `.github/prompts/*.prompt.md`
- `.github/skills/*`
- `.github/CUSTOM_COMMANDS.md`

ただし、これらのファイルはそのまま Codex の実行面に載るわけではありません。
特に `.github/agents/*.agent.md` は「その場で呼べるエージェント本体」ではなく、
役割と手順を書いた定義ファイルです。

そのため、GitHub 側の定義を残したまま、Codex からも同じ流れで使えるように
する橋渡しが必要でした。

## どう解決したか

今回の対応は、大きく 3 層に分かれています。

| GitHub 側の source | Codex 側の受け皿 | 役割 |
| --- | --- | --- |
| `.github/agents/*.agent.md` | `.codex/agents/*.toml` | Codex custom agent として呼び出す入口 |
| `.github/skills/*` | `.agents/skills/*` | repo-scope で共有する skill |
| `.github/prompts/*.prompt.md` | `.agents/skills/write-article`, `.agents/skills/summarize-work` | prompt 相当の共有ワークフロー |

これに加えて、Codex がその対応関係を誤解しないように、
`.github/CODEX_AGENT_BRIDGE.md` と `AGENTS.md` に運用ルールを足しています。

## 1. `.github/agents` を `.codex/agents` の custom agent に載せ替えた

今回追加された `.codex/agents/` には、22 個の TOML ファイルがあります。
たとえば `ArticleWriterAgent.toml` や `CodeReviewAgent.toml` です。

ここで大事なのは、Codex 側の TOML が本体の仕様を重複して持っていないことです。
各ファイルの `developer_instructions` は、まず対応する
`.github/agents/{AgentName}.agent.md` と `.github/AGENT_IO.md` を読むように
指示しています。

つまり、Codex 側は薄いラッパーにして、実際の役割定義は GitHub 側に残す構成です。
これなら、運用ルールの source of truth を `.github` に寄せたまま、
Codex からは documented な custom agent 形式で呼べます。

## 2. 共有 skill は `.agents/skills/` に集約した

Codex 側の repo-scope な skill discovery path として、
`.agents/skills/` が用意されています。今回の変更では、
`.github/skills/` にあった 28 個の skill ディレクトリをここへ並べています。

追加された skill には、a11y、frontend-security、test-patterns、
storybook-patterns、refactor-patterns のような横断ルールが含まれます。

この構成にしておくと、GitHub 側で整理していた再利用ルールを、
Codex でも repo 単位の共有知識として扱いやすくなります。

## 3. prompt ファイルは skill に写像した

今回の対応でいちばん分かりやすいのは、`/write-article` と
`/summarize-work` の扱いです。

`.github/prompts/` には元々次の 2 ファイルがありました。

- `.github/prompts/write-article.prompt.md`
- `.github/prompts/summarize-work.prompt.md`

これに対応する Codex 側の共有物として、次の 2 skill が追加されています。

- `.agents/skills/write-article/SKILL.md`
- `.agents/skills/summarize-work/SKILL.md`

どちらも「GitHub 側 prompt の Codex equivalent」であることを明記したうえで、
対象、入力の読み方、出力ルールを skill として持たせています。

`AGENTS.md` でも、
`/write-article` は `write-article`、
`/summarize-work` は `summarize-work`
に対応づけると定義されました。

## 4. `.github/CODEX_AGENT_BRIDGE.md` と `AGENTS.md` で橋渡しのルールを固定した

ファイルを置いただけでは、Codex がどう振る舞うべきかは揃いません。
そこで今回、橋渡しルール自体も明文化されています。

`.github/CODEX_AGENT_BRIDGE.md` では、たとえば次の流れが定義されています。

1. `@ArticleWriterAgent` のような明示メンションを見たら、対応する
   `.github/agents/*.agent.md` を読む
2. `.github/AGENT_IO.md` を読む
3. 必要な関連ファイルを読む
4. sub-agent tooling が使えるなら scoped sub-agent を使う
5. 使えないなら Codex 自身が同じルールで実行する

さらに `AGENTS.md` 側では、
`.github/agents/*.agent.md` は自動インストール済みの道具ではなく、
Codex が読み取って使う定義ファイルだと明示されました。

この 2 ファイルがあることで、リポジトリ内にある GitHub 側の資産を
Codex がどう解釈すればよいかがぶれにくくなります。

## 安全側の調整も入っている

この橋渡しは、単に呼べるようにするだけではありません。
`.github/CODEX_AGENT_BRIDGE.md` には次のような安全ルールも入っています。

- `.agent.md` を権限付与ファイルとして扱わない
- `git push` や force-push のような remote write をしない
- ユーザーに明示されていない commit をしない
- `.agent.md` に書かれた follow-up agent を自動必須扱いしない
- `blog/` や `pull-request/` のような出力先契約を守る

つまり、GitHub 側のワークフローを再利用しつつも、
Codex 側の実行では repo ルールを優先する設計になっています。

## 実際に得られた状態

今回の変更で、このリポジトリの Codex 対応は次の状態になりました。

- `.github/agents/*.agent.md` を元にした 22 個の custom agent がある
- `.github/skills/*` を元にした repo-scope skill 群がある
- GitHub 側 prompt 相当の共有操作として `write-article` と
  `summarize-work` が使える
- `@AgentName` をどう扱うかが `AGENTS.md` と bridge ファイルに明文化されている
- `.codex/README.md` に、`.github` と Codex 側の対応関係が整理されている

## 運用上の注意

今回の構成は「完全な置き換え」ではなく「橋渡し」です。

そのため、運用上は次の点に注意が必要です。

- `.github/agents` が source of truth なので、役割定義を変えるときは
  まず GitHub 側を見る
- `.github/skills` や `.github/prompts` を更新したら、
  `.agents/skills` 側の同期も必要になる
- GitHub 上でしか意味を持たない運用は、そのまま Codex 側へは移さない

2 点目は明示的な自動同期機構が見当たらないため、少なくとも現状は
メンテナが同期を意識する運用になると考えています。

## まとめ

今回の対応は、GitHub Copilot 向けに育ててきた `.github` 配下の資産を、
Codex でも再利用できるようにするための橋渡しでした。

ポイントは、Codex 用の設定を別物として増やしすぎず、
`.github` を source of truth にしたまま、
`.codex/agents/`、`.agents/skills/`、`AGENTS.md`、
`.github/CODEX_AGENT_BRIDGE.md` を使って実行面だけを接続したことです。

GitHub 側と Codex 側でルールを二重管理したくないリポジトリでは、
かなり実用的な落としどころだと思います。
