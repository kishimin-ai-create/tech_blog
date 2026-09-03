# Codex CLI更新時の配布元フォールバック警告を抑制する

## 結論

Codex CLIの更新で`releases.openai.com`のチェックサム取得に失敗し、GitHub Releasesへフォールバックする警告が表示された。ユーザー環境変数`CODEX_INSTALLER_USE_RELEASES_OPENAI_COM=0`を設定して配布元をGitHub Releasesへ固定し、Codex CLI 0.153.0への更新を警告なしで完了できた。

## 発生した問題

更新コマンドの実行時に次の警告とエラーが発生した。

```text
WARNING: Could not download or verify https://releases.openai.com/codex/releases/0.153.0/codex-package_SHA256SUMS; retrying from GitHub Releases.
The term 'Get-FileHash' is not recognized as the name of a cmdlet...
```

インストーラーは既定で`releases.openai.com`を試し、取得または検証に失敗するとGitHub Releasesへ切り替える。今回の実行では、そのフォールバック経路で`Get-FileHash`の解決にも失敗した。

## 解決方法

インストーラーが公式配布元へ接続できない環境でも、最初からGitHub Releasesを使うようにユーザー環境変数を設定した。

```powershell
[Environment]::SetEnvironmentVariable(
  'CODEX_INSTALLER_USE_RELEASES_OPENAI_COM',
  '0',
  'User'
)
```

新しいPowerShellプロセスで更新を実行する。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command '$env:CODEX_NON_INTERACTIVE="1"; irm https://chatgpt.com/codex/install.ps1 | iex'
```

## 動作確認

更新結果は次のとおりだった。

```text
Codex CLI 0.153.0 installed successfully.
```

さらに`codex --version`で`codex-cli 0.153.0`を確認し、同じコマンドを再実行してフォールバック警告が出ないことを確認した。

## 注意点

- この設定が抑制するのは、`releases.openai.com`からGitHub Releasesへ切り替える際の警告である。
- GitHubへの接続失敗や別の検証エラーまで解決するものではない。
- 既に開いているシェルにはユーザー環境変数が反映されない場合があるため、新しいプロセスで確認する。
- 既存の`Get-FileHash`切り分け手順は、コマンドレット自体が見つからない場合の調査に引き続き有効である。

## まとめ

今回の警告は、優先配布元のチェックサム取得失敗後にフォールバックしたことが直接の契機だった。インストーラーの配布元選択を環境変数で明示し、更新経路を安定させたうえでバージョン確認まで行うのがポイントである。

## 参考資料

- [Codex CLIのインストールで`Get-FileHash`が見つからないときの切り分け](./01-codex-get-filehash-install-error.md)
