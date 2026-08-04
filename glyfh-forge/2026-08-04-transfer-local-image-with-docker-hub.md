# Docker DesktopのローカルイメージをDocker Hub経由で別PCへ移す

Docker Desktop内だけにあるイメージは、別PCのDocker Desktopへ自動では同期されない。しかし、Docker Hubへpushすれば、クラウド事業者固有のコンテナRegistryを用意せずに別PCからpullできる。

この記事では、Windows上の`glyph-forge:local`を非公開のDocker Hubリポジトリへ保存し、別PCで取得してFastAPIを呼び出すまでを扱う。コンテナのボリュームや実行中データの移行は対象外とする。

## 検証環境

2026年8月4日に次の条件で確認した。

- ホストOS: Windows
- Docker Desktop / Docker Engine: 29.6.2
- イメージ: `glyph-forge:local`
- イメージのOS・アーキテクチャ: `linux/amd64`
- イメージサイズ: 110,971,472 bytes
- コンテナ内の待受ポート: 8080

Docker Hubへのpush、別名タグのmanifest確認、コンテナ起動、`/health`と`/images`へのHTTPリクエストを実行した。

## Docker DesktopだけではPC間同期にならない

Docker DesktopのImages画面に表示されるローカルイメージは、そのPCのDocker Engineが管理している。別PCでも同じDockerアカウントへサインインしただけでは、ローカルイメージは現れない。

PC間で移す方法は大きく2つある。

1. `docker save`でtarアーカイブへ書き出し、別PCで`docker load`する
2. Docker HubなどのRegistryへpushし、別PCでpullする

USBメモリや共有ストレージで単発移行するならtarが単純である。複数PCから繰り返し取得するなら、Registry経由のほうが更新と配布を管理しやすい。Docker公式にも、[`docker image save`](https://docs.docker.com/reference/cli/docker/image/save/)と[`docker image push`](https://docs.docker.com/reference/cli/docker/image/push/)の両方が用意されている。

## Docker Hubへ非公開リポジトリを作る

公開リポジトリへpushすると誰でもイメージを取得できる。ソースコードやフォントなど、公開を想定していない内容がイメージへ含まれる可能性があるため、今回は非公開リポジトリを使用した。

Docker Hubへサインインし、My HubのRepositoriesから次のリポジトリを作る。

- Namespace: 自分のDocker ID
- Repository name: `glyph-forge`
- Visibility: Private

画面操作は[Docker公式のリポジトリ作成手順](https://docs.docker.com/docker-hub/repos/create/)で確認できる。プランごとの利用可能数や制限は変わる可能性があるため、作成時にDocker Hubの表示を確認する。

## 元PCからイメージをpushする

Docker DesktopでDocker Hubへサインインしたうえで、PowerShellからDocker IDを変数へ設定する。`YOUR_DOCKER_ID`は実際のDocker IDへ置き換える。

```powershell
$dockerId = 'YOUR_DOCKER_ID'
docker image tag glyph-forge:local "${dockerId}/glyph-forge:local"
docker image push "${dockerId}/glyph-forge:local"
```

ローカルタグの`glyph-forge:local`だけでは、Dockerはpush先のNamespaceを判断できない。`docker image tag`で`Docker ID/リポジトリ名:タグ`の形式を追加してからpushする。

今回のpushでは全レイヤーが`Pushed`となり、次のdigestが返った。

```text
sha256:cc6e3d871641c33b2fe86ce07b21bd918ebdac45cf9b0b3bf9d1e9d17879c431
```

digestは同じイメージ内容を識別する値である。タグは付け替えられるが、digestを記録しておけば取得対象の確認に使える。

## 別PCでpullして起動する

別PCのDocker Desktopでも、非公開リポジトリを読めるDocker Hubアカウントへサインインする。その後、PowerShellでpullする。

```powershell
$dockerId = 'YOUR_DOCKER_ID'
docker image pull "${dockerId}/glyph-forge:local"
```

Docker DesktopのImages画面にイメージが表示されたら、ローカルホストだけへ8080番を公開して起動する。

```powershell
$dockerId = 'YOUR_DOCKER_ID'
docker container run --detach `
  --name glyph-forge-api `
  --publish 127.0.0.1:8080:8080 `
  "${dockerId}/glyph-forge:local"
```

`127.0.0.1:8080:8080`と指定すると、同じPCからだけAPIへ接続できる。`8080:8080`のようにホストIPを省略するとLAN側から到達できる場合があるため、外部アクセスが不要ならループバックアドレスへ限定する。

## APIを呼び出して移行を確認する

最初にヘルスチェックを呼ぶ。

```powershell
curl.exe http://127.0.0.1:8080/health
```

検証時はHTTP 200と次のJSONが返った。

```json
{"status":"ok"}
```

次に画像生成APIへJSONを送信し、応答をPNGファイルへ保存する。

```powershell
curl.exe --request POST http://127.0.0.1:8080/images `
  --header "Content-Type: application/json" `
  --data '{"frame_text":"TEST","inner_text":"INNER","outer_text":"OUTER"}' `
  --output output.png
```

検証時はHTTP 200、Content-Type `image/png`、21,056 bytesの応答を確認した。入力内容によってPNGのサイズは変わるため、バイト数の一致ではなく、HTTPステータスとContent-Type、保存した画像の内容を確認する。

FastAPIの対話的なAPIドキュメントは次のURLで開ける。

```text
http://127.0.0.1:8080/docs
```

## うまくいかない場合の確認

### `requested access to the resource is denied`

タグのNamespaceがサインイン中のDocker IDまたは所属Organizationと一致しているか確認する。非公開リポジトリでは、pullする側にも読み取り権限が必要になる。

```powershell
docker image ls glyph-forge
docker image ls YOUR_DOCKER_ID/glyph-forge
```

### `unknown hostname`

`example.sakuracr.jp`のような別事業者のRegistry名を付けただけでは、Registry自体は作成されない。過去のログイン情報がDocker Desktopへ残っていても、接続先Registryが現存する証明にはならない。Docker Hubを使う場合は`YOUR_DOCKER_ID/glyph-forge:local`へタグを付け直す。

### APIへ接続できない

コンテナの状態とポート割り当てを確認する。

```powershell
docker container ps --filter "name=glyph-forge-api"
docker container logs --tail 50 glyph-forge-api
```

ホストの8080番がすでに使用中なら、ホスト側だけ別の番号へ変えられる。

```powershell
docker container run --detach `
  --name glyph-forge-api `
  --publish 127.0.0.1:18080:8080 `
  YOUR_DOCKER_ID/glyph-forge:local
```

この場合のAPI URLは`http://127.0.0.1:18080`になる。

## コンテナを停止・撤去する

確認後にコンテナを残さない場合は、停止して削除する。Docker Hub上のイメージと、ローカルのイメージは削除されない。

```powershell
docker container stop glyph-forge-api
docker container rm glyph-forge-api
```

## まとめ

Docker Desktopはローカルイメージを別PCへ直接同期しないが、Docker HubをRegistryとして使えば`push`と`pull`で移せる。今回の検証では、非公開リポジトリへ保存した`linux/amd64`イメージを取得し、8080番でコンテナを起動して、ヘルスチェックとPNG生成APIまで確認できた。

単発のオフライン移行には`docker save`と`docker load`、継続的な配布にはDocker Hubというように使い分けるとよい。イメージだけではボリュームや実行時データは移らないため、状態を持つコンテナでは別途バックアップ方法を設計する必要がある。

## 参考資料

- [Docker Desktopでイメージを操作する](https://docs.docker.com/desktop/use-desktop/images/)
- [Docker Hubリポジトリを作成する](https://docs.docker.com/docker-hub/repos/create/)
- [`docker image push`リファレンス](https://docs.docker.com/reference/cli/docker/image/push/)
- [`docker image save`リファレンス](https://docs.docker.com/reference/cli/docker/image/save/)
