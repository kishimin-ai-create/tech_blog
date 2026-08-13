# WindowsからAppRun向けlinux/amd64イメージを作る

## はじめに

開発PCがWindowsでも、デプロイ先と同じOS・CPUアーキテクチャのコンテナイメージを作る必要がある。Glyph Forgeのデプロイ手順では、さくらのクラウドAppRun共用型に合わせて`linux/amd64`を明示した。

## ビルド対象をコマンドに書く

ローカル用のビルドは次の形になる。

```powershell
docker build --platform linux/amd64 -t glyph-forge:local .
```

`--platform linux/amd64`を省略してホスト任せにすると、別のCPUアーキテクチャの開発PCへ移ったときに、同じ手順から異なる成果物が作られる可能性がある。デプロイ先の条件をコマンドへ含めることで、手順を再現しやすくした。

## ローカルタグは退避にも使える

`glyph-forge:local`はDocker Desktop内のローカルイメージであり、クラウド上のレジストリとは別物である。開発中の起動確認には、8080番を公開して使う。

```powershell
docker run --rm -p 8080:8080 --name glyph-forge-local glyph-forge:local
```

別のターミナルから`/health`と`/images`を確認すれば、コンテナとしての起動経路と画像生成経路を通せる。

## レジストリでは変更不能なタグを使う

デプロイ用には`latest`だけに頼らず、GitコミットSHAをタグに使う。どのコードから作ったイメージかを追跡でき、問題が起きたときに直前のイメージへ戻しやすくなる。

手順書では、ローカルビルド、レジストリ用タグ付け、pushを次の順序にしている。

1. Git SHAを取得する
2. `linux/amd64`でビルドする
3. `<registry>.sakuracr.jp`配下の名前を付ける
4. 認証後にpushする

認証パスワードはコマンド引数、Dockerfile、Git管理ファイルへ書かない。AppRunが非公開レジストリから取得する場合も、push用とpull用の権限を可能な範囲で分ける。

## 関連する成果物

- `docs/sakura-cloud-deployment.md`
- `Dockerfile`
- `README.md`
- コミット `8a2e29d`、`08983ff`

## まとめ

認証パスワードはコマンド引数、Dockerfile、Git管理ファイルへ書かない。AppRunが非公開レジストリから取得する場合も、push用とpull用の権限を可能な範囲で分ける。
