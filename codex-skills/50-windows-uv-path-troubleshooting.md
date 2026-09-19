---
title: "Windowsでuvが認識されないときにユーザーPATHを確認する"
tags: [Windows, PowerShell, uv]
---

# Windowsでuvが認識されないときにユーザーPATHを確認する

Windowsで`uv`をインストール済みなのにPowerShellから実行できない場合、実行ファイルの有無とPATHの反映状態を分けて確認します。本記事では、収集処理の実行時に発生した`uv: The term 'uv' is not recognized`を、既存のPATHを壊さずに切り分けた手順を記録します。

## 結論

原因は、`uv.exe`が存在する一方で、現在のPowerShellプロセスのPATHにそのディレクトリが含まれていなかったことです。ユーザー環境変数への登録後も、既に起動済みのPowerShellには変更が自動反映されません。

## 1. 実行ファイルを直接確認する

まず、インストール先の`uv.exe`を直接呼び出します。バージョンが表示されれば、インストール自体は成功しています。

```powershell
& "<uv.exeのディレクトリ>\uv.exe" --version
```

## 2. ユーザーPATHとプロセスPATHを分けて確認する

PowerShellから次を実行すると、ユーザー環境変数と現在のプロセス環境変数を比較できます。

```powershell
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
$processPath = $env:Path
```

ユーザーPATHに登録済みでも、現在のPowerShellの`$env:Path`が古ければ`uv`は見つかりません。

## 3. 現在のPowerShellへ反映する

既存の値を保持したまま、ユーザーPATHとマシンPATHを現在のプロセスへ読み込み直します。

```powershell
$env:Path = [Environment]::GetEnvironmentVariable("Path", "User") + ";" + [Environment]::GetEnvironmentVariable("Path", "Machine")
uv --version
```

別の方法として、PowerShellを閉じて新しく開く方法でも反映できます。

## 4. 収集コマンドを実行する

`uv`が認識された後は、プロジェクトルートで次を実行します。

```powershell
uv run python -m app.collector collect-backfill
```

## 注意点

- `.env.development`を使う場合は、実行前にその環境変数を読み込む必要がある。
- PATH登録はユーザー環境変数へ限定し、システム全体のPATHは変更しない。
- `uv`が認識されても、DB接続や外部HTTP取得が成功したことまでは意味しない。終了コードと収集結果を別途確認する。

今回の確認では、`uv 0.12.15`を実行ファイルへ直接指定して起動できることを確認し、ユーザーPATHには重複登録しない条件で登録状態を確認しました。
