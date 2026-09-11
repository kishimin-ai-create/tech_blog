# VS Codeの既存ウィンドウへフォルダーを追加・削除する

## はじめに

アプリケーション本体とドキュメントのように、別々の場所にある複数のフォルダーを同じVisual Studio Codeウィンドウで扱いたいことがあります。

VS CodeのMulti-root Workspaceを使うと、フォルダーを移動したり同じ親ディレクトリへまとめたりせず、1つのExplorerへ複数のroot folderを表示できます。この記事では、PowerShellから既存のVS Codeウィンドウへフォルダーを追加し、不要なフォルダーをワークスペースから外す方法を説明します。

## 完成形

次の2つのコマンドで、最後に操作したVS Codeウィンドウのroot folderを変更できます。

```powershell
code --add "C:\work\product"
code --remove "C:\work\old-documentation"
```

`--add`はフォルダーをMulti-root Workspaceへ追加します。`--remove`はワークスペースから登録を外すだけで、ディスク上のフォルダーやファイルを削除しません。

## 前提条件

- OS: Windows 11
- Shell: PowerShell
- VS Code: `1.137.0`で動作確認
- `code`コマンドをPowerShellから実行できること
- 追加先となるVS Codeウィンドウが開いていること

この記事で使用するオプションは、VS Code `1.137.0`の`code --help`と公式CLIドキュメントの両方で確認しました。別バージョンでは実行前にhelpを確認してください。

## 手順

### Step 1: CLIが利用できることを確認する

最初に、PowerShellでVS Code CLIのバージョンとhelpを確認します。

```powershell
code --version
code --help
```

今回確認した環境では、次のオプションが表示されました。

```text
-a --add <folder>     Add folder(s) to the last active window.
--remove <folder>     Remove folder(s) from the last active window.
```

`code`が見つからない場合は、VS Codeの実行ファイルまたはCLIが`PATH`から利用できる状態かを先に確認します。

### Step 2: フォルダーを既存ウィンドウへ追加する

追加対象を絶対パスで指定します。パスに空白が含まれる可能性を考え、二重引用符で囲みます。

```powershell
code --add "C:\work\documentation"
```

`--add`は最後に操作したVS Codeウィンドウへフォルダーを追加します。Explorerには、もともと開いていたフォルダーと追加したフォルダーがそれぞれrootとして表示されます。

2つ目のフォルダーを単一フォルダーのウィンドウへ追加すると、VS CodeはUntitled Workspaceを作成します。ウィンドウを閉じた後も明示的なファイルとして再利用したい場合は、VS Codeの「File > Save Workspace As...」で`.code-workspace`ファイルを保存します。

### Step 3: 不要なフォルダーをワークスペースから外す

不要になったroot folderは`--remove`で外します。

```powershell
code --remove "C:\work\old-documentation"
```

この操作の対象はVS Codeのワークスペース構成です。フォルダー自体を削除するコマンドではありません。ワークスペース整理のためにファイル削除を行う必要はありません。

### Step 4: 追加先のウィンドウを確認する

`--add`と`--remove`は「最後に操作したウィンドウ」を対象にします。VS Codeを複数ウィンドウで開いている場合は、変更したいウィンドウを一度操作してからコマンドを実行します。

実行後は、そのウィンドウのExplorerでroot folderの一覧を確認します。意図しないウィンドウへ追加した場合は、同じパスを`--remove`へ渡して外し、対象ウィンドウを操作してから`--add`をやり直せます。

## UIから操作する方法

CLIを使わず、VS Codeの画面からも同じ構成を変更できます。

- 追加: 「File > Add Folder to Workspace...」
- 削除: Explorerのroot folderを右クリックし、「Remove Folder from Workspace」

一度だけ手動で選ぶ場合はUIが分かりやすく、正確なパスを再利用したい場合や、セットアップ手順として残したい場合はCLIが扱いやすい方法です。

## 動作確認

この記事の作成前に、実在する2つのフォルダーを使って次と同じ形式の操作を実行しました。

```powershell
code --remove "C:\work\old-documentation"
code --add "C:\work\documentation"
```

記事では個人のローカルパスを`C:\work\...`へ置き換えています。両コマンドは終了コード`0`で完了し、成功時に標準出力を返しませんでした。

CLIの提供オプションは次のコマンドでも再確認しました。

```powershell
code --help | Select-String -Pattern '^  -a --add|^  --remove' -Context 0,1
```

コマンドの終了だけでは、意図したウィンドウが変更されたことまでは証明できません。最後にExplorerのroot folder一覧を目視で確認してください。

## 注意点

### フォルダーを開く操作との違い

次のコマンドはフォルダーを開く操作であり、既存のMulti-root Workspaceへ追加する意図を明示していません。

```powershell
code "C:\work\documentation"
```

既存ウィンドウへの追加を目的とする場合は`--add`を使用します。

### Workspace Trust

信頼済みのMulti-root Workspaceへ新しいフォルダーを追加すると、VS Codeがそのフォルダーを信頼するか確認する場合があります。内容を確認できないフォルダーは安易に信頼せず、Restricted Modeで開いて確認します。

### ワークスペース設定の保存場所

保存済みMulti-root Workspaceの全体設定は`.code-workspace`ファイルに置かれます。一方、各root folderはそれぞれの`.vscode`設定を持てます。設定の適用範囲が異なるため、複数リポジトリを並べただけで各リポジトリの設定ファイルが統合されるわけではありません。

## まとめ

VS Codeの既存ウィンドウへフォルダーを追加するには`code --add`、ワークスペースから外すには`code --remove`を使います。

```powershell
code --add "C:\work\documentation"
code --remove "C:\work\old-documentation"
```

どちらも最後に操作したVS Codeウィンドウが対象です。`--remove`はディスク上のファイルを削除しません。複数のroot folder構成を継続して使う場合は、`.code-workspace`として保存しておくと再度開きやすくなります。

## 参考資料

- [Visual Studio Code: Command Line Interface](https://code.visualstudio.com/docs/configure/command-line)
- [Visual Studio Code: Multi-root Workspaces](https://code.visualstudio.com/docs/editing/workspaces/multi-root-workspaces)
- [Visual Studio Code: What is a VS Code workspace?](https://code.visualstudio.com/docs/editing/workspaces/workspaces)
- [Visual Studio Code: Workspace Trust](https://code.visualstudio.com/docs/editing/workspaces/workspace-trust)

確認日: 2026年9月11日
