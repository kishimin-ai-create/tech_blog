# Claude CodeだけにあるSkillをCodexでも使えるようにする

Claude CodeとCodexを併用していると、片方にだけ追加したSkillがもう片方から見えなくなる。本記事では、Claude側にしかなかったSkillを、共有Skillの正本へ安全に取り込み、Codexから利用可能にした手順を説明する。

対象読者は、`$HOME/.claude/skills`と`$HOME/.agents/skills`をローカルで管理している開発者である。ここでは`$HOME/.agents/skills`を正本とし、コピー元だけにあるディレクトリを削除しない。

## 先に役割を決める

今回の配置は次の役割で運用した。

| パス | 役割 |
| --- | --- |
| `$HOME/.agents/skills` | CodexとClaude Codeで共有するSkillの正本 |
| `$HOME/.claude/skills` | Claude Codeが参照するSkill配置 |
| `$HOME/.codex/skills/.system` | Codexが同梱するシステムSkill |

正本を決めずに双方向コピーすると、同名Skillのどちらが新しいか判断できなくなる。今回はClaude限定Skillを一度正本へ「取り込む」操作と、正本から各クライアントへ「同期する」操作を区別した。

## ディレクトリ名を比較する

最初に、双方のトップレベルSkillディレクトリを比較する。

```powershell
$agentNames = @(
  Get-ChildItem -LiteralPath "$HOME\.agents\skills" -Directory |
    ForEach-Object Name
)

$claudeNames = @(
  Get-ChildItem -LiteralPath "$HOME\.claude\skills" -Directory |
    ForEach-Object Name
)

Compare-Object -ReferenceObject $agentNames -DifferenceObject $claudeNames
```

`=>`はClaude側だけ、`<=`は正本側だけに存在することを表す。今回、Claude側だけにあったのは`skill-creator`と`techblog-diary`だった。

## 名前だけでコピー対象を決めない

`skill-creator`は正本側にはなかったが、CodexのシステムSkillとして既に利用可能だった。これをユーザーSkillとして重複コピーすると、同名Skillの由来と更新経路が曖昧になる。そのためコピー対象から除外した。

残った`techblog-diary`について、次を確認した。

1. `SKILL.md`と付属ファイルの一覧
2. シンボリックリンクや意図しない生成物の有無
3. APIキー、トークン、パスワード、秘密鍵の有無
4. ユーザー名を含む固定パスの有無
5. Codexから参照する相対リンクが解決できるか

確認の結果、ファイルは`SKILL.md`だけで、秘密情報はなかった。ただし、保存先にユーザー固有の絶対パスが含まれていた。

## 変更前にコピーをプレビューする

コピー先が存在しないことを確認し、`-WhatIf`で対象を表示する。

```powershell
$claudeSkill = "$HOME\.claude\skills\techblog-diary"
$agentSkill = "$HOME\.agents\skills\techblog-diary"

if (Test-Path -LiteralPath $agentSkill) {
  throw "Destination already exists: $agentSkill"
}

Copy-Item -LiteralPath $claudeSkill -Destination $agentSkill -Recurse -WhatIf
```

表示されたコピー元とコピー先が意図どおりなら、同じコマンドから`-WhatIf`を外して実行する。この操作は既存の同名ディレクトリを上書きせず、Claude側のファイルも削除しない。

## 正本へ取り込む際に固定パスを除く

ユーザーHOME配下の参照へユーザー名を埋め込むと、別環境でSkillを再利用できない。取り込み後、次のように`$HOME`基準へ変更した。

```text
# 変更前
C:\Users\<USER>\Desktop\programming\TechBlog\...

# 変更後
$HOME\Desktop\programming\TechBlog\...
```

相対参照の`../write-article/SKILL.md`は、正本側にも`write-article`が存在するため、そのまま利用できた。

## Skillを検証する

Codex同梱のSkill検証スクリプトをUTF-8モードで実行した。

```powershell
$env:PYTHONUTF8 = '1'
python "$HOME\.codex\skills\.system\skill-creator\scripts\quick_validate.py" `
  "$HOME\.agents\skills\techblog-diary"
```

結果は`Skill is valid!`だった。Claude版との差分も確認し、意図した固定パスの正規化1箇所だけであることを確認した。

新しいCodexセッションでは、Skill一覧に`techblog-diary`が現れ、`$techblog-diary`として呼び出せた。

## 同期と取り込みを混同しない

正本からClaude側へ通常同期するときは、既存の同期スクリプトを`-WhatIf`付きで実行してから本実行する。一方、今回のようなClaude限定Skillの救出では、対象をレビューしてから正本へ個別に取り込む。

完全ミラーのための削除オプションは、コピー先だけにあるSkillを消す可能性がある。明示的に全削除対象を確認できない限り使わない。

## 残っている作業

取り込んだ`techblog-diary`はCodexから利用可能になったが、作業時点では正本リポジトリに未コミットだった。また、Claude側には元の固定パスが残っている。今後は次の順序で整える。

1. 正本側のSkillだけを限定してレビュー・コミットする。
2. 正本からClaude側へ同期し、両者の内容を一致させる。
3. 再度、相対パスとファイル内容を比較する。

確認日: 2026年8月11日
