# Codex CLIのインストールで`Get-FileHash`が見つからないときの切り分け

WindowsでCodex CLIをインストールした際、チェックサム検証で`Get-FileHash`が見つからず終了した事例を整理する。対象は、PowerShellの基本操作ができ、公式インストールスクリプトを使っている開発者である。

結論から言うと、今回の直接原因は確定していない。現在の端末ではWindows PowerShell 5.1とPowerShell 7.6.4の双方で`Get-FileHash`を利用でき、元のエラーを再現できなかった。したがって、PowerShell 7からの再実行は回避候補であり、検証済みの根本解決ではない。

## 発生した症状

Codex CLI 0.147.0のインストール中、次の順序で処理が停止した。

1. `releases.openai.com`からチェックサムを取得または検証できなかった。
2. インストーラーがGitHub Releasesへフォールバックした。
3. `Get-FileHash`を解決できず、終了コード1になった。

主要なエラーは次の内容だった。

```text
The term 'Get-FileHash' is not recognized as the name of a cmdlet,
function, script file, or operable program.
```

公式リポジトリのWindowsインストーラーは、アーカイブのSHA-256を`Get-FileHash`で計算して期待値と比較する。また、`releases.openai.com`を優先し、取得や検証に失敗した場合はGitHub Releasesへ切り替える実装になっている。

## 最初に確認すること

PowerShellで次のコマンドを実行する。

```powershell
$PSVersionTable.PSVersion
Get-Command Get-FileHash
Get-Module -ListAvailable Microsoft.PowerShell.Utility
```

期待する結果は、PowerShellのバージョンと`Get-FileHash`のコマンド情報が表示されることである。`Get-FileHash`が見つからない場合は、インストーラーを再実行する前にPowerShell本体と`Microsoft.PowerShell.Utility`の解決状態を確認する。

複数のPowerShellが入っている場合は、実体も確認する。

```powershell
where.exe powershell
where.exe pwsh
```

`powershell.exe`はWindows PowerShell、`pwsh.exe`はPowerShell 7以降を指す。ターミナル上で確認した環境と、別プロセスから起動された環境が同じとは限らない。

## PowerShell 7から再実行する

`pwsh`で`Get-FileHash`を解決できる場合は、PowerShell 7のプロンプトから公式コマンドを実行する。

```powershell
$env:CODEX_NON_INTERACTIVE = '1'
irm https://chatgpt.com/codex/install.ps1 | iex
```

別のシェルからPowerShell 7へ渡す場合は、次の形式を使える。

```powershell
$env:CODEX_NON_INTERACTIVE = '1'
irm https://chatgpt.com/codex/install.ps1 | pwsh -NoProfile -Command -
```

インターネットから取得したスクリプトをそのまま実行するため、実行前にURLと取得内容がOpenAI公式のものであることを確認する。監査が必要な環境では、スクリプトを保存・レビューしてから実行する。

## インストール結果を確認する

再実行後は、別のPowerShellウィンドウを開いて確認する。

```powershell
Get-Command codex
codex --version
```

今回の調査では再インストールを実行していない。そのため、この手順で元の環境が復旧することは未検証である。再び失敗した場合は、実行したPowerShellのバージョン、`Get-Command Get-FileHash`の結果、インストーラーの最初の失敗箇所を保存して比較する。

## 分かったことと分からなかったこと

確認できた事実は次のとおりである。

- 2026年8月11日時点の端末にはPowerShell 7.6.4とWindows PowerShell 5.1があった。
- 確認時には双方で`Get-FileHash`を解決できた。
- 公式インストーラーはチェックサム検証に`Get-FileHash`を使っていた。
- GitHub Releasesへのフォールバックは、インストーラーが備える通常の代替経路だった。

元の失敗時だけコマンドを解決できなかった理由は分かっていない。PowerShellの版だけを原因と断定せず、呼び出された実体、プロファイルの有無、モジュール探索パス、親プロセスから渡された環境を分けて確認する必要がある。

## 参考資料

- [OpenAI Codex公式リポジトリのインストール手順](https://github.com/openai/codex#installing-and-running-codex-cli)
- [OpenAI Codex公式Windowsインストーラー](https://github.com/openai/codex/blob/main/scripts/install/install.ps1)

確認日: 2026年8月11日
