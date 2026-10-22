# FastAPIのAPIバージョンをインストール済みパッケージと揃える

## 結論

glyph-forgeの第6回アップロードを1.0.5として識別できるよう、パッケージの版を更新し、FastAPIにはインストール済み配布物のバージョンを渡した。APIと配布物の版を同じ値として運用するプロジェクトで、二重管理を避けるための変更である。

確認日は2026年10月22日。対象はPython 3.12、FastAPI、setuptoolsで配布するglyph-forgeだった。APIの互換性バージョンをパッケージとは別に管理する設計には、そのまま適用しない。

## 発生した問題

`setup.py`には1.0.0が残り、`FastAPI()`で作成したAPIのOpenAPI文書は0.1.0を返していた。利用者から6回目のアップロードに対応する1.0.5を指定されたが、どちらの値もそのリリースを表していなかった。

これは画像生成の品質とは別の問題である。版が揃っても、本番でどのイメージが動いているかまでは証明できない。

## 調査と回帰テスト

HTTP経由で`/openapi.json`を取得し、`info.version`とインストール済み配布物のメタデータを比較するテストを追加した。

```python
from importlib.metadata import version

from fastapi.testclient import TestClient

from app.main import app


def test_openapi_reports_the_installed_package_version():
    client = TestClient(app)

    response = client.get("/openapi.json")

    assert response.status_code == 200
    assert response.json()["info"]["version"] == version("glyph_forge")
```

修正前は0.1.0と1.0.0の不一致で失敗した。設定ファイルの文字列を検査するだけではなく、クライアントが取得するOpenAPIの値を確認した。

## 解決方法

`setup.py`のバージョンを1.0.5へ更新し、APIの初期化時に次の値を渡した。

```python
from importlib.metadata import version

from fastapi import FastAPI

app = FastAPI(version=version("glyph_forge"))
```

値の参照元はソース上の別の定数ではなく、実際にインストールされた配布物である。ローカルではメタデータを更新するため、既存の開発環境で`pip install --no-deps -e .`を実行した。本番用Dockerfileはwheelをインストールする構成で、そのイメージでも確認した。

## 動作確認

| 検証 | 観測結果 |
| --- | --- |
| OpenAPIの回帰テスト | Redの不一致を確認後、Green |
| Windowsの全体テスト | 152 passed、5 skipped |
| autoflake、isort、Black、flake8、mypy | 成功 |
| wheel・sdistのビルド | 1.0.5の配布物を生成 |
| linux/amd64の本番用イメージ | パッケージ・OpenAPIとも1.0.5、Raqm有効 |

| 全体カバレッジ（appとsrc/glyph_forge） | 結果 |
| --- | ---: |
| 文・行 | 90.74% |
| 分岐 | 80.60% |
| 総合 | 89.03% |

5件のスキップはWindowsで利用できないRaqm関連4件とLinux実行要件1件だった。Linuxの結果をWindowsのカバレッジへ合算してはいない。

## 運用上の制約

この実装は、配布物がインストールされていることを前提とする。ソースを置くだけの実行環境ではメタデータを取得できず、起動に失敗しうる。今回の配布・コンテナの経路ではインストールを確認したが、未インストールでの起動を支援する変更は行っていない。

APIとパッケージの版を分ける方針へ変える場合は、単純な一致テストも見直す必要がある。また、このテストは両者の一致を固定するもので、リリース番号の選び方自体を検証するものではない。

## 根拠

- [PR #6：版の統一、実行した検証、配布イメージ](https://github.com/kishimin/glyph-forge/pull/6)
- [実装コミット031d675](https://github.com/kishimin/glyph-forge/commit/031d675ffc5a5cc7afda5a084e9211ae4c40f0a8)
- [回帰テストのコミット2b44dd1](https://github.com/kishimin/glyph-forge/commit/2b44dd1)
