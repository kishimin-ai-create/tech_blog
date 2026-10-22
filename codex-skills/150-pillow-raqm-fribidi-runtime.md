# 絵文字フォントを同梱しても、Pillowの結合絵文字には実行時の文字組み機能が必要だった

## 結論

glyph-forgeのLinuxコンテナでは、絵文字用フォントを同梱しただけではRaqmが有効にならなかった。`libfribidi0`を実行用イメージへ追加するとRaqmが利用可能になり、結合絵文字を含む39件の新規描画テストがすべて成功した。

対象は、Pillowを使う画像生成サービスをslimイメージで動かす人である。2026年10月22日に、`python:3.12.11-slim-bookworm`、Pillow 10.3.0、Linux amd64の実行用イメージで確認した。個々の絵文字に対するフォント選択は[149番の記事](./149-pillow-grapheme-emoji-font-selection.md)に分けた。

## 発生した問題

### 症状

フォントを追加したイメージで新規テストを実行すると、通常の描画テストは成功したが、結合絵文字の4ケースがスキップされた。

```text
31 passed, 4 skipped
```

この時点のテストファイルは35ケースだった。後から増えた最終39ケースの結果とは区別する。

### 発生条件と影響

スキップ条件は`features.check_feature("raqm")`が偽になることだった。Raqmは、複数の文字を適切な字形へ組み立てる複雑な文字組みをPillowから利用するための機能である。通常の絵文字が描けても、ZWJで結合する絵文字や国旗の合成まで確認できたことにはならなかった。

## 調査

### 実行用イメージの機能を直接確認する

プロジェクトルートでビルドしたイメージを対象に、次を実行した。

```powershell
docker run --rm glyph-forge:issue-5 python -c "from PIL import features; assert features.check_feature('raqm'), 'Combined emoji require Raqm and FriBiDi'"
```

修正前は`AssertionError`になった。ホストのPythonだけで調べず、配布に使う実行用イメージで不足を確かめた。

### Pythonパッケージ以外の依存を調べる

[Pillow 10.3.0の公式ビルド資料](https://raw.githubusercontent.com/python-pillow/Pillow/10.3.0/docs/installation/building-from-source.rst)では、RaqmとFriBiDiの関係、および標準wheelでFriBiDiを実行時に読み込む仕組みが説明されている。今回の観測結果も、実行用イメージにFriBiDiが必要であることを示していた。

## 原因

使用したslimイメージにはFriBiDiがなく、インストールしたPillowのRaqm機能を実行時に利用できなかった。フォントファイルの同梱確認と、文字組み機能の利用可能性は別の確認項目だった。

## 解決方法

マルチステージビルドの実行用ステージに、次を追加した。builderだけに追加しても、runtimeへライブラリが渡るとは限らない。

```dockerfile
RUN apt-get update \
    && apt-get install --no-install-recommends -y libfribidi0 \
    && rm -rf /var/lib/apt/lists/*
```

この変更はイメージのビルド時にDebianパッケージを取得する。稼働中コンテナへ手作業で追加する方法にはしなかった。

## 動作確認

```powershell
docker build --platform linux/amd64 -t glyph-forge:issue-5 .
```

ビルド後のイメージへテストファイルと`pytest.ini`を読み取り専用でマウントし、一時コンテナにpytest 7.4.4を入れて、次を実行した。

```text
python -m pytest -o addopts= -q /checks/test_emoji_rendering_medium.py
39 passed in 5.26s
```

新規テストは、LinuxでRaqmが利用可能なことを明示的に要求する。👩‍🚀、🇯🇵、👍🏽、1️⃣は、Raqmの選択、単一絵文字と同じ送り幅、一つのセルへの描画を検証した。これはテストしたシーケンスの確認であり、全Unicodeシーケンスの合成を保証するものではない。

さらに、非rootユーザーで起動したローカルコンテナへHTTP要求を送り、通常画像・アイコン・背景画像の3経路で200 PNG応答を確認した。テスト用コンテナへpytestを入れたことと、実行イメージにpytestを追加することは区別した。

## 再発防止

Windowsのローカル環境でもRaqmが利用できず、結合絵文字の4件とLinux専用要件の1件はスキップした。それを成功数へ含めず、Linuxの実行イメージで5件とも実行できたことを確認した。

本番と同じイメージの確認は、デプロイ済みサービスの確認とは異なる。今回確認したのはローカルでビルド・起動したイメージであり、デプロイは実施していない。WindowsでのFriBiDi導入も未実施である。

## まとめ

フォントが存在すること、文字組み機能が有効なこと、APIがPNGを返すことを順に検証した。slimイメージの依存不足は、Pythonパッケージのインストール成功や、条件付きテストの成功数だけでは見落とし得る。

## 根拠

- [修正後のDockerfile](https://github.com/kishimin/glyph-forge/blob/31b31f8e8e3136a5808cbcdce8cdd056558c5575/Dockerfile)
- [Raqm要件と結合絵文字のテスト](https://github.com/kishimin/glyph-forge/blob/31b31f8e8e3136a5808cbcdce8cdd056558c5575/tests/test_glyph_forge/test_emoji_rendering_medium.py)
- [実装時の検証記録](https://github.com/kishimin/glyph-forge/blob/31b31f8e8e3136a5808cbcdce8cdd056558c5575/review/issue-5-emoji-rendering.md)
