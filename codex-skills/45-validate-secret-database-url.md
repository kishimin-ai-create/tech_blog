# Pydanticの`SecretStr`だけではDB URLを検証できない

## 結論

`SecretStr`は値の表示を隠す型であり、空文字やURL形式までは検証しない。DB接続先を起動時に検証するには、`MySQLDsn`のような用途別の型を別途使う必要がある。

さらに、URL検証で発生した`ValidationError`をそのまま構造化ログへ渡すと、入力値に含まれる認証情報が残り得る。slope-collectorでは、MySQL URLの検証と、実行時設定を返す`load_settings()`でのエラー入力除去を組み合わせた。

## 発生した問題

### 症状

設定クラスでは`DATABASE_URL`を必須の`SecretStr`としていた。しかし、次の2値でも設定の生成に成功した。

```text
DATABASE_URL=
DATABASE_URL=not-a-database-url
```

一方、`MySQLDsn`による検証を単純に追加すると、不正なURLを拒否できても、`ValidationError.json()`の`input`へ接続文字列が残った。

### 発生条件

- OS: Windows 11
- Python: 3.14.7
- Pydantic: 2.13.5
- pydantic-settings: 2.15.0
- 対象: 環境変数から読み込むMySQL接続URL

### 影響

- 空または不正なURLのままAPIを起動できる
- 実際の失敗がmigrationやDBアクセスまで遅れる
- 不正URLの例外を構造化して記録すると、ユーザー名やパスワードを含む入力値がログへ出る可能性がある

## 調査

### `SecretStr`の責務を確認する

最初に、空文字と通常の文字列を環境変数へ設定し、`load_settings()`が失敗するかをテストした。修正前は、どちらも`ValidationError`を発生させなかった。

この結果から、`SecretStr`が担当するのは秘匿表示であり、接続URLとしての妥当性ではないことを確認した。

### URL検証時のエラーを確認する

次に、別DBのschemeと識別用文字列を含むURLを渡した。通常の例外表示だけでなく、構造化ログを想定して`ValidationError.json()`も調べた。

```python
assert sentinel not in str(error.value)
assert sentinel not in error.value.json()
```

URL形式を拒否するだけの実装では、2つ目のassertionが失敗した。Pydanticのvalidation errorは、原因調査のために元入力を保持するためである。

## 原因

問題は2つの責務を1つとして扱ったことにあった。

1. `SecretStr`は通常の`repr`で値をマスクする。
2. `MySQLDsn`はschemeやURL構造を検証する。
3. `ValidationError`は診断用に不正入力を保持する。

したがって、`SecretStr`へ型を変更するだけではURL検証にならず、URL検証を追加するだけでは例外の全出力経路を秘匿できない。

## 解決方法

設定フィールドは既存コードとの互換性を保つため`SecretStr`のままとし、field validatorでMySQL URLとして検証した。

```python
@field_validator("database_url")
@classmethod
def validate_database_url(cls, value: SecretStr) -> SecretStr:
    try:
        MySQLDsn(value.get_secret_value())
    except ValidationError:
        message = "database URL must be a valid MySQL URL"
        raise ValueError(message) from None
    return value
```

低レベルのURLエラーをそのまま伝播させず、安全なメッセージへ変換する。さらに、アプリケーションが環境変数を読む唯一の入口である`load_settings()`では、再構築した`ValidationError`から元入力を除外した。

この境界により、呼び出し側は従来どおり`ValidationError`を扱いつつ、例外文字列とJSONのどちらにも接続URLを残さない。

## 動作確認

```text
uv run ruff format --check .        -> pass
uv run ruff check .                 -> pass
uv run mypy app tests               -> pass
uv run pytest --cov=app             -> 8 passed, 100%
uv run pre-commit run --all-files   -> pass
docker build --tag slope-collector:review . -> pass
```

回帰テストでは、次を確認した。

- 有効な`mysql+pymysql` URLを受理する
- URLがない場合に拒否する
- 空文字とURLでない文字列を拒否する
- 不正URLの例外文字列とJSONに識別用文字列が含まれない

## 再発防止

- 秘密値の型と、値の形式を検証する型の責務を分ける
- 例外の`str()`だけでなく、ログ基盤が利用し得るJSON表現もテストする
- 実行時設定は`load_settings()`を経由し、`Settings`を直接生成する経路を増やさない
- exampleファイルには実資格情報を置かず、初回セットアップでローカルファイルへコピーしてから置換する

## 制約

`MySQLDsn`は`mysql+pymysql`以外のMySQLドライバーschemeも受理する。今回の要件はMySQL URLの検証であり、ドライバーをPyMySQLだけへ限定する判断は行っていない。

また、入力除去は`load_settings()`の公開境界で実施している。今後、環境変数から`Settings`を生成する別経路を追加する場合は、同じ秘匿契約を適用する必要がある。

## まとめ

- `SecretStr`は秘匿表示を担当し、URL形式は検証しない
- DB接続先は用途別DSN型で起動時に検証する
- validation errorは元入力を保持し得るため、文字列と構造化表現の両方を確認する
- 秘密設定を読む入口を1つに集約すると、検証とエラー秘匿を同じ境界で保証しやすい

確認日: 2026年9月18日
