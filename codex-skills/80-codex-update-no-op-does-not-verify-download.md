# Codex CLIの同一バージョン更新成功ではダウンロードを検証できない

## はじめに

codex update が成功と表示されても、すでにインストール済みの同じバージョンを確認しただけなら、配布物のダウンロードやチェックサム検証を通ったとは限らない。この記事では、Windowsで更新エラーを復旧した後に同じコマンドが成功したケースを使い、「コマンドの成功」と「更新経路の検証」を分けて記録する。

## 前提・環境

- OS: Windows 11
- Codex CLI: 0.156.1から0.157.1へ更新
- Windows PowerShell: 5.1.26100.9444
- PowerShell 7: 7.6.6

## やってみた結果

codex update は最終的に終了コード0で成功した。ただし、その時点でCLIはすでに0.157.1だった。出力にはダウンロードがなく、公式インストーラーの実装も完全な既存リリースを検出するとダウンロード処理を省略する。そのため、この成功は将来の新バージョンをダウンロードして検証できることの証明にはならない。

## 実装・検証

### Step 1: 更新コマンドの失敗を記録する

更新前の codex --version は codex-cli 0.156.1だった。通常の codex update は0.157.1を検出してダウンロードを開始したが、チェックサム検証中に Get-FileHash を解決できず、終了コード1になった。

~~~powershell
codex update
~~~

エラーは次の内容だった。

~~~text
Get-FileHash : The term 'Get-FileHash' is not recognized as the name of a cmdlet, function, script file, or operable program.
Error: ... install.ps1 ... failed with status exit code: 1
~~~

この出力だけでは、その実行環境でコマンドが解決されなかった理由までは分からない。

### Step 2: PowerShell 7から公式インストーラーを実行する

pwsh が利用できたため、公式インストーラーをPowerShell 7へ渡して更新した。

~~~powershell
$env:CODEX_NON_INTERACTIVE = '1'
irm https://chatgpt.com/codex/install.ps1 |
  pwsh -NoProfile -ExecutionPolicy Bypass -Command -
~~~

インストーラーは Codex CLI 0.157.1 installed successfully. と出力した。新しいPowerShellプロセスで codex --version を実行し、codex-cli 0.157.1を確認した。codex doctor は19 ok、1 idle、7 notes、5 warn、0 failを報告した。

### Step 3: 同じバージョンで通常の更新コマンドを再実行する

~~~powershell
codex update
~~~

このコマンドも終了コード0で成功したが、インストーラーは Downloading Codex CLI を出力しなかった。実行時点で0.157.1がすでにインストール済みだったためである。

2026年9月27日に確認した公式Windowsインストーラーでは、既存リリースの内容が完全だと判定した場合、アーカイブのダウンロードとチェックサム検証へ進まない。この挙動から、同一バージョンでの再実行結果は、最初の失敗箇所が修復されたことを単独では示さないと判断できる。

## つまずいたところ

### 問題

通常の更新コマンドではチェックサム計算時に Get-FileHash が見つからなかった。その後の通常コマンドは成功したものの、すでに最新の同じバージョンだったため、問題となったダウンロード経路を再実行していなかった。

### 原因

今回の端末で最初に Get-FileHash が解決できなかった根本原因は確定していない。PowerShell 7のPSModulePathをWindows PowerShell 5.1の子プロセスが継承することで同様のエラーが起きる報告が、Codex公式リポジトリにある。ただし、今回の実行時の子プロセス環境は保存されておらず、この端末の原因だとは確認できていない。

## 学んだこと

更新コマンドの終了コードだけでなく、更新前後のバージョンと、ダウンロード・検証処理が実際に行われたかを合わせて記録する。同一バージョンを対象にした成功は、インストーラーの既存リリース判定を通った結果かもしれない。

## まとめ

Codex CLIを0.156.1から0.157.1へ更新し、別プロセスからバージョンとDoctorの結果を確認した。一方、復旧後に実行した通常の codex update は同じバージョンを検出しただけで、ダウンロードとチェックサム検証を再実行した証拠にはならなかった。次のバージョンで通常更新が成功するかは未確認である。

## 参考資料

- [Codex公式Windowsインストーラー](https://github.com/openai/codex/blob/main/scripts/install/install.ps1)
- [Windows standalone updateでPSModulePathがpowershell.exeへ継承される報告](https://github.com/openai/codex/issues/27117)
- [Get-FileHashが利用できないWindows更新の報告](https://github.com/openai/codex/issues/46684)
- [関連する切り分け記事](./01-codex-get-filehash-install-error.md)

確認日: 2026年9月27日
