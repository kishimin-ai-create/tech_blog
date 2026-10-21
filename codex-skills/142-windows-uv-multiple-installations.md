# Windowsのuv更新ではPATH上の複数インストールを確認する

## はじめに

2026年10月22日、Windows上のuvを更新した。最初に見つかった実行ファイルだけでなく、PATH上にあった通常インストール版とWinGet版の両方を確認した記録である。

## 前提・環境

- Windows、PowerShell。
- 更新前のuvは両方とも0.12.15。
- 更新先は当日の公式最新版0.13.4。

## やってみた結果

2つのインストールをそれぞれの更新方法で0.13.4へ更新した。新しいPowerShellで、両方のパスのuvとuvxが0.13.4を返すことを確認した。

## 実装・検証

### 実行ファイルを列挙する

```powershell
uv --version
Get-Command uv -All | Select-Object Source
where.exe uv
```

優先されていたのは`$HOME/.local/bin/uv.exe`で、その後ろにWinGet管理のuvも存在した。通常の`uv --version`だけでは、後者の版を確認できない。

### インストール方法ごとに更新する

通常インストール版は、その実行ファイルを指定して更新した。WinGet版にはパッケージ管理側の更新を使った。以下はPC上のツールを書き換える操作である。

```powershell
& "$HOME/.local/bin/uv.exe" self update
winget upgrade --id astral-sh.uv --exact --silent --disable-interactivity
```

どちらも0.12.15から0.13.4へ更新された。WinGetではダウンロードしたアーカイブのハッシュ検証も成功した。

### 新しいプロセスでも確認する

```powershell
pwsh -NoProfile -Command 'uv --version; uvx --version; Get-Command uv -All | ForEach-Object { $_.Source; & $_.Source --version }'
```

実際の確認ではuvxも全パスを列挙した。uv・uvxともに2か所すべてが0.13.4だった。

## つまずいたところ

ホスト側の更新後も、slope-collectorは`required-version = "==0.12.24"`を要求していた。そのためプロジェクト操作は版の不一致で停止した。ホストの成功を、プロジェクトやDockerの更新完了とは扱えなかった。

プロジェクト要求とコンテナ版を揃える仕組みは、[既存のuv更新記事](./120-uv-container-update-required-version.md)にまとめている。今回は両方を0.13.4へ変更し、ロック済み依存関係を維持したまま検証した。

## 学んだこと

更新の確認単位はコマンド名だけでは決まらない。PATHが異なる端末やジョブでは、別の実行ファイルが選ばれる可能性がある。今回確認できたのは、列挙した2つのインストールと、新しいPowerShellからの解決結果である。

uv 0.13.4は、問題のあるemSignルート証明書を同梱CAから除去する修正を含む。これはPHPで別途公表された脆弱性とは異なる問題であり、この更新だけで他のソフトウェアの証明書ストアまで更新されたことにはならない。

## 参考資料

- [uv 0.13.4のリリース](https://github.com/astral-sh/uv/releases/tag/0.13.4)
- [uvのセキュリティアドバイザリ](https://github.com/astral-sh/uv/security/advisories/GHSA-hq85-5cpj-mr62)
- [uv公式の更新方法](https://docs.astral.sh/uv/getting-started/installation/#upgrading-uv)
