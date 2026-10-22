# GitHubに載せるAPI URLは、読者の使い方から決める

## 結論

APIリポジトリのGitHubへURLを載せるときは、読者がAPIを呼び出したいのか、完成した製品を使いたいのかで案内先を分ける。glyph-forgeでは、外部開発者に公開APIとして案内するなら、AboutのWebsiteをAPIドキュメントへ向ける方針を提案した。

2026年10月22日に実際の設定を確認すると、Websiteには既にAPIの`/docs`が掲載されていた。ドキュメントのHTTP応答とOpenAPIの版を読み取った結果も含め、提案と確認済みの状態を分けて記録する。

## 背景

glyph-forgeは、繰り返す文字で文字形の画像を作るPythonライブラリとFastAPIサービスである。Mojicaは、その画像生成を利用するWeb製品である。利用者から「glyph-forgeのGitHubにAPIのURLを載せるのは適切か」と質問された。

APIを呼びたい開発者と、画像を作りたい製品利用者では、最初に必要な説明が異なる。URLの掲載場所だけを決める前に、誰への入口なのかを整理した。

## 比較した案内先

相談では次の条件付きの方針を示した。外部向けAPIとして提供するか自体を、この相談で確定したわけではない。

| 読者の目的 | AboutのWebsiteの候補 | READMEで補う説明 |
| --- | --- | --- |
| 公開APIを直接利用する | APIドキュメント | ベースURL、呼び出し例、利用制限 |
| Mojicaで画像を作る | Mojicaの画面 | 製品とAPIの関係 |

APIドキュメントを入口にすると、開発者がリクエストとレスポンスの説明へ進める。一方、製品を操作したい読者には、Mojicaの画面へ直接進める導線が合う。これは今回の案内方針であり、どちらかを全リポジトリへ一律に適用する規則ではない。

## 公開状態を読み取りで確認した

まずリポジトリのメタデータを取得した。

```powershell
gh repo view kishimin/glyph-forge --json description,homepageUrl
```

`homepageUrl`は、AppRun上のAPIドキュメント`/docs`を指していた。掲載されているだけで到達できるとは限らないため、そのURLをHTTP GETし、同じAPIの`/openapi.json`も取得した。

| 確認対象 | 観測結果 |
| --- | --- |
| GitHubのWebsite設定 | APIドキュメントを指している |
| ドキュメントのHTTP応答 | 200 |
| ドキュメントのHTML | Swagger UIの初期化を含む |
| OpenAPIの`info.version` | 1.0.5 |

この記録作業ではWebsiteやREADMEを変更していない。誰がいつWebsiteを設定したかも確認していないため、自分が掲載した成果としては扱わない。

## どこまで確かめたか

今回確かめたのは、掲載先がドキュメントとして応答し、APIの版が取得できることまでである。HTTP GETによる確認なので、ブラウザー上でSwagger UIの全機能を操作した結果ではない。

ドキュメントが公開されていることだけでは、外部利用のサポート範囲や利用条件まで確定しない。READMEへ利用制限などを案内する提案はしたが、今回それらを新設したわけではない。

画像生成の動作についても、この確認では再実行していない。別途実施した[公開サイトの生成画像の検証](./153-public-generated-image-pixel-verification.md)とは、確認した内容が異なる。

## 再利用する判断

AboutのWebsiteは、リポジトリを見つけた読者が最初に進む入口として選ぶ。READMEには、その入口だけでは不足する呼び出し方や利用条件を置く。今回のようにAPIと製品が分かれている場合は、両者の関係を説明したうえで、主な読者に合わせて案内する。

- [glyph-forgeのリポジトリと現在のAbout](https://github.com/kishimin/glyph-forge)
- [製品として画像を生成するMojica](https://mojica.pages.dev/)
- [同日の相談と読み取り確認の日誌](https://github.com/kishimin-test/tech_blog/blob/main/python/glyph-forge/20261022/20261022.md)

URLやAPIの応答は確認日の状態である。実装変更を伴わない設計相談の記録であり、公開APIの提供契約を決定するADRの代わりにはしない。
