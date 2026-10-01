# WindowsでCodexの余分なターミナルを`--no-daemon`で切り分ける

## はじめに

WindowsでCodex CLIを使うと、操作中に別のターミナルウィンドウが開くことがある。今回は`codex --no-daemon`で別ウィンドウが出なくなった、という利用者の観察を起点に、オプションの意味と公開された類似事例を確認した。

この記事で分かるのは、`--no-daemon`が切り替える起動経路と、現時点で確認できる原因の範囲である。

## 前提・環境

- 確認日: 2026年10月1日
- OS: Windows（今回のCLI確認環境）
- CLI: `codex-cli 0.157.1`（`codex --version`の出力）
- 観察された症状: `codex --no-daemon`を使うと、以前開いていた別のターミナルウィンドウが開かなくなった

## やってみた結果

手元の`codex --help`は、`--no-daemon`を「共有バックグラウンドサーバーが動作中でも、それを使わず実行する」オプションと説明していた。Codexの公開Issueにも、通常起動では追加のコンソールウィンドウが開き、`--no-daemon`では開かないというWindowsでの報告がある。

したがって、今回の表示差は共有サーバーを経由しないことと整合する。ただし、今回のPCでプロセスツリーを採取していないため、どの補助プロセスがウィンドウを開いていたかは確定していない。

## 実装・検証

### CLIのオプションを確認する

PowerShellで次を実行した。

```powershell
codex --version
codex --help
```

`codex --version`は`codex-cli 0.157.1`を返した。`codex --help`には次の説明が表示された。

```text
--no-daemon  Run without the shared background server, even if it is already running
```

このオプションは、別ウィンドウを直接隠す表示設定ではなく、Codexが使うサーバーの起動経路を変更する。

### 公開された類似事例を照合する

[OpenAIのCodex Issue #48090](https://github.com/openai/codex/issues/48090)には、Windowsで通常の`codex`を起動すると追加のコンソールウィンドウが開き、`codex --no-daemon`では開かないという報告がある。報告者は、共有のマネージドサーバーから起動された補助プロセスと`conhost.exe`をプロセスツリーで確認している。

このIssueは類似症状の根拠であり、今回のPCのプロセスを直接調べた結果ではない。

## つまずいたところ

最初の「出なくなった」という説明だけでは、エラーメッセージ、Codexの画面、別のターミナルウィンドウのどれが消えたのか特定できなかった。対象が別ウィンドウであると確認してから、`--no-daemon`の意味と公開Issueを照合した。

## 学んだこと

- FACT: 手元のCLIヘルプでは、`--no-daemon`は共有バックグラウンドサーバーを使わない設定である。
- FACT: 利用者は別のターミナルウィンドウが開かなくなったと報告した。
- FACT: 公開Issue #48090には、同じオプションで追加ウィンドウが開かなくなるWindowsの事例がある。
- INFERENCE: 今回の表示差も、共有サーバーを経由しなくなったことによる可能性が高い。
- 未確認: 今回のPCでウィンドウを開いていたプロセスの名前と親子関係。

## まとめ

`codex --no-daemon`で別ウィンドウが開かなくなったという利用者の報告は、共有バックグラウンドサーバーを使わない起動経路への変更と整合する。個別の原因プロセスまで確かめる場合は、通常起動時にプロセスツリーとウィンドウの対応を採取する必要がある。

## 参考資料

- [OpenAI Codex Issue #48090: Windows managed daemon opens two visible console windows](https://github.com/openai/codex/issues/48090)
