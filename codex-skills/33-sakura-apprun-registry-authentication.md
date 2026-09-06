# さくらのクラウドAppRunでコンテナレジストリ認証に失敗したときの切り分け

## 結論

AppRunの`image authorization failed`は、イメージが存在しないとは限らず、非公開コンテナレジストリへAppRunが認証できていない状態を示す。レジストリ専用ユーザーにPull権限を与え、AppRunのレジストリ認証へ同じ認証情報を入力する必要がある。

## 発生した問題

Mojica APIのイメージを次へpushした後、AppRunでアプリケーションを作成しようとすると、次のエラーになった。

```text
Validation Error
image authorization failed
```

## 調査

ローカルでは次のpullが成功した。

```text
glyph-forge-public.sakuracr.jp/mojica-api:20260906
Digest: sha256:127994f3a743bf2564b13b71840806ce305e7801222eadeb108086dcaa556ca8
Status: Image is up to date
```

この結果から、イメージ名とタグはレジストリに存在することを確認できた。したがって、AppRun側の認証設定を確認する必要があった。

## 原因

レジストリは非公開設定だったため、AppRunがイメージをpullするにはレジストリ専用ユーザーの認証情報が必要だった。ローカルDockerのログイン状態はAppRunへ共有されない。

## 解決方法

レジストリの「ユーザ」タブでユーザーを作成し、権限を`Push & Pull`または`All`にする。AppRunでは次を設定する。

```text
コンテナイメージ:
glyph-forge-public.sakuracr.jp/mojica-api:20260906

レジストリ認証:
利用する
```

ユーザー名とパスワードには、さくらのクラウドのログイン情報ではなく、レジストリ専用ユーザーの値を入力する。パスワードを再設定した場合は、ローカルでも再ログインしてpullを確認する。

## 参考資料

- [さくらのクラウド コンテナレジストリ](https://manual.sakura.ad.jp/cloud/appliance/container-registry/index.html)
- [AppRun共用型クイックスタート](https://manual.sakura.ad.jp/cloud/apprun/getting_started.html)
