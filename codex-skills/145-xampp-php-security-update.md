# XAMPPのPHPを設定と拡張を維持してセキュリティ更新する

## はじめに

2026年10月22日、Windows上のPHPを8.2.12から8.2.35へ更新した。対象はPATHから呼ばれるXAMPP付属のPHPである。バージョン表示だけでなく、設定、既存拡張、Apacheモジュールの読込を確認した。

## 前提・環境

- PHPの実体は`C:/xampp/php/php.exe`。
- x64、Thread Safe、Visual C++ 2019ビルド。
- Apacheは更新時点で起動していなかった。
- PCOVなど、配布ZIP以外に由来する拡張があった。

## やってみた結果

新しいPowerShellでもPHP 8.2.35が解決された。既存の`php.ini`のハッシュと読み込まれる拡張の集合は維持された。Apacheの`httpd -t`は`Syntax OK`を返し、`httpd -M`では`php_module (shared)`を確認した。

## 実装・検証

### 版と導入元を確認する

```powershell
php -v
Get-Command php -All | Select-Object Source
php --ini
php -m
```

PHP 8.2.12はXAMPP配下にあり、Apache設定も同じディレクトリの`php8ts.dll`と`php8apache2_4.dll`を参照していた。

公式の10月21日告知では、CVE-2026-109618とCVE-2026-109619に対する8.2系の修正版は8.2.35だった。互換性の確認範囲を抑えるため、同じ8.2系・x64・Thread Safeの公式Windows配布を選んだ。

### 配布物と候補環境を検証する

公式ページからZIPを取得し、そのページに記載されたSHA-256と照合した。まず配布物の`php.exe -n -v`で、既存設定なしに起動できることを確認した。

次に、既存PHPディレクトリのコピーへ新しい配布ファイルを重ねた候補を作った。候補側の`php.ini`と`extension_dir`を指定して起動し、元のモジュール一覧と比較した。PCOV、PDO SQLite、OpenSSLのロードとJSON処理の簡単な実行も成功した。

### 退避してから切り替える

対象パスがXAMPP直下であることと、Apache・PHPの対象プロセスが起動していないことを再確認した。旧ディレクトリを退避し、候補を元の`php`パスへ切り替えた。切り替え後の確認に失敗した場合に旧ディレクトリへ戻す経路も用意した。

## つまずいたところ

既存ディレクトリには追加ファイルが多く、コピーには時間がかかった。処理途中で更新完了とは報告せず、候補の起動と拡張確認が終わってから切り替えた。

## 学んだこと

CLIとApacheが同じPHPを参照する構成では、PATHの`php -v`だけで確認を終えない。今回はCLI、設定ハッシュ、拡張一覧、Apacheの構文とモジュール読込、主要3ファイルと配布物のハッシュ一致を確認した。

WebアプリのHTTP応答や全機能テストは実施していない。旧版の退避ディレクトリも残しているため、PC内のPHPバイナリをすべて修正版へ置き換えたという結果ではない。確認したのは現在選択されるXAMPPのPHPである。

## 参考資料

- [2026年10月21日のPHP公式セキュリティ告知](https://www.php.net/archive/2026.php#2026-10-21-5)
- [PHP 8.2の公式Windows配布](https://www.php.net/downloads.php?os=windows&version=8.2)
