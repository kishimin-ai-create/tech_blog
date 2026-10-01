# PowerShellでCodexを毎回`--no-daemon`で起動する

## はじめに

WindowsでCodex CLIの入力中に黒い別ウィンドウが開いた。以前に`codex --no-daemon`で別ウィンドウが開かなくなったとの利用者報告があり、今回はPowerShellで普段どおり`codex`と入力しても同じオプションを渡す設定を追加した。

この記事は、その設定と確認できた範囲を記録する。`--no-daemon`で起動経路がどう変わるかは[別記事](./86-codex-no-daemon-windows-terminal.md)で扱う。

## 完成形

PowerShellのユーザープロファイルに次の関数を置く。`codex`というコマンド名は維持し、CLI実行時に`--no-daemon`を渡す。

```powershell
function codex {
    $codexExecutable = (Get-Command codex.exe -CommandType Application).Source
    & $codexExecutable --no-daemon @args
}
```

`codex.exe`を明示して実行すれば、この関数を通さずに起動できる。

## 前提条件

- 確認日: 2026年10月1日
- OS: Windows 11
- シェル: PowerShell 7.6.6
- Codex CLI: 更新前`0.157.1`、設定確認時`0.159.3`
- 対象: PowerShellから`codex`と入力して起動する場合

## 設定手順

### Step 1: ユーザー共通プロファイルを確認する

PowerShellで以下を実行する。

```powershell
$PROFILE.CurrentUserAllHosts
Test-Path $PROFILE.CurrentUserAllHosts
```

今回の環境では、PowerShell 7のユーザー共通プロファイルは未作成だった。パスは各自の環境で取得し、ユーザー名を固定したパスを設定に埋め込まない。

### Step 2: `codex`関数を保存する

プロファイルが既にある場合は内容を確認し、既存設定を残して関数を追加する。今回の環境では新規ファイルを作成し、「完成形」の関数を保存した。

関数は`Get-Command codex.exe`で実行ファイルを探し、`@args`で利用者の引数を渡す。`--no-daemon`は共有バックグラウンドサーバーを使わない起動オプションであり、ウィンドウ表示を直接制御する設定ではない。

### Step 3: 新しいPowerShellで読み込みを確認する

```powershell
Get-Command codex | Select-Object Name, CommandType
codex --version
codex update --help
```

今回の確認では`codex`の`CommandType`は`Function`、版数は`codex-cli 0.159.3`だった。`codex update --help`も表示され、サブコマンドへの引数転送を確認した。CLI本体の実行ファイルを指定する`Get-Command codex.exe`も解決できた。

## 動作確認の範囲

対話画面の自動起動検証は、実行環境のPTY生成エラーで失敗した。新しいPowerShellでCodexを操作し、以前の再現操作である`@t`入力時に別ウィンドウが開かないかは、利用者の確認待ちである。現時点で「症状が解消した」とは判定しない。

設定が不要になったら、ユーザー共通プロファイルから上記の`codex`関数だけを削除する。プロファイル内の他の設定は残す。

## まとめ

- FACT: 更新後のCLIは`0.159.3`で、新しいPowerShellは`codex`関数を読み込んだ。
- FACT: `codex --version`と`codex update --help`は関数経由で実行できた。
- INFERENCE: 普段のPowerShell起動時にも`--no-daemon`が渡される構成になった。
- 未確認: 対話操作時に黒い別ウィンドウが開かなくなったかどうか。

## 参考資料

- [OpenAI Codex Issue #48090](https://github.com/openai/codex/issues/48090)
