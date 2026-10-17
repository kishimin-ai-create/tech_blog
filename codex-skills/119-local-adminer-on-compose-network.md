# 起動済みMySQLへローカル限定のAdminerを接続する

## はじめに

APIのSwagger UIを確認した後、DBもブラウザで見たいという依頼があった。既存MySQLを使い、同じDockerネットワークへAdminerを追加した。ここでは管理画面のHTTP応答と、管理画面コンテナからDBへの到達確認を分けて記録する。

## 完成形

Windowsホストの`http://localhost:8081`でAdminerのログイン画面が開く。MySQLの接続先には、コンテナ間で解決できる`db`を指定する。管理画面の公開ポートは`127.0.0.1`へ限定した。

## 前提条件

- Docker Desktopと既存のCompose環境が起動している。
- MySQLサービス名は`db`、ネットワーク名は`slope-collector_default`。
- DB名とユーザーは`collector`。パスワードはローカル設定で管理する。
- 同名のAdminerコンテナがないこと、8081が使えることを事前に確認する。

これらは今回の環境の名前であり、別プロジェクトへそのまま適用しない。

## Implementation

### Step 1: DBとネットワークを確認する

新しいDBを作らず、既存コンテナの起動状態とネットワークを確認した。

```powershell
docker ps --format '{{.Names}} {{.Ports}}'
docker inspect slope-collector-db-1 --format '{{range $name, $config := .NetworkSettings.Networks}}{{$name}}{{end}}'
```

DBは起動済みで、ネットワークは`slope-collector_default`だった。既存の認証情報を使ったSQL照会でも、DB名とテーブル一覧を取得できた。パスワードをコマンド出力へ表示していない。

### Step 2: Adminerを追加する

プロジェクトルートから次を実行した。これはイメージを取得し、新しいコンテナとローカル公開ポートを作る操作である。

```powershell
docker run -d --name slope-collector-adminer --network slope-collector_default --publish 127.0.0.1:8081:8080 --env ADMINER_DEFAULT_SERVER=db --restart unless-stopped adminer:5
```

今回はComposeファイルを変更せず、補助コンテナとして追加した。`adminer:5`は可変タグであり、同じ内容を再現する必要がある運用では固定方法を別途決める。

### Step 3: ログイン設定を確認する

ログイン画面では、種類をMySQL、サーバーを`db`、ユーザーとデータベースを`collector`にする。パスワードは既存の`.env.development`の`MYSQL_PASSWORD`を参照する。記事へパスワードは転載しない。

## 動作確認

```powershell
Invoke-WebRequest http://127.0.0.1:8081 -TimeoutSec 15
docker exec slope-collector-adminer php -r '$s = @fsockopen("db", 3306, $errno, $errstr, 5); if (!$s) { exit(1); } fclose($s); echo "MySQL reachable\n";'
```

管理画面はHTTP 200、TCP接続は成功した。別途、MySQLコンテナ内でSQLも実行できた。ただし、Adminerのフォームへ認証情報を入力してログイン完了まで操作したわけではない。HTTP 200とTCP到達だけで、管理画面の認証成功まで主張しない。

## 撤去と制約

今回追加した補助コンテナだけを撤去する場合は、対象名を確認してから`docker rm -f slope-collector-adminer`を実行する。MySQLのボリュームを削除する必要はない。この撤去コマンドは記事執筆時には実行していない。

API再構築の後にも、管理画面のHTTP 200と`db:3306`への到達を再確認した。認証情報の変更や本番公開は対象外である。

## 根拠

2026年10月17日の`docker inspect`、`docker run`、`Invoke-WebRequest`、`fsockopen`による確認ログと、対象プロジェクトの`compose.yaml`に基づく。
