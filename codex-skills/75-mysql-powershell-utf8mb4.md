---
title: "PowerShell経由のMySQL更新で日本語が文字化けしたときの切り分け"
tags: [MySQL, PowerShell, UTF-8, トラブルシュート]
---

# PowerShell経由のMySQL更新で日本語が文字化けしたときの切り分け

## 結論

MySQLへ日本語を投入した後にアプリの検索が一致しない場合、接続の文字コードだけでなく、保存済みバイト列そのものを確認する。`HEX()`で値を調べ、二重エンコードなら正しいUTF-8バイト列へ修正し、アプリ接続には`charset=utf8mb4`を指定する。

## 発生した症状

DBの表示上は日本語に見える値を設定したが、PythonのSQLAlchemy接続では設定値との一致件数が0になった。接続結果を表示すると、アプリ側では文字化けした文字列になっていた。

## 調査

まず、接続の文字コードを確認した。

```sql
SELECT @@character_set_client,
       @@character_set_connection,
       @@character_set_results,
       HEX(name)
FROM sources;
```

接続設定を`utf8mb4`にしても、`HEX(name)`の値が期待する日本語のUTF-8列と一致しなかった。つまり、今回の主因は読み出し時だけでなく、更新時に保存された値が二重エンコードされていたことだった。

## 解決方法

対象行を主キーで限定し、正しいUTF-8バイト列を`utf8mb4`として更新した。

```sql
UPDATE sources
SET name = CONVERT(0x<UTF8_BYTES> USING utf8mb4)
WHERE id = <TARGET_ID>;
```

また、アプリの接続URLへ文字コードを指定した。

```text
mysql+pymysql://user:password@host:3306/database?charset=utf8mb4
```

更新後、PythonのSQLAlchemy接続から設定値で検索し、対象行の一致件数が1になることを確認した。

## 再発防止

- 日本語を含むDB更新では、表示結果だけでなく`HEX()`も確認する。
- PowerShell、mysqlクライアント、Python接続のそれぞれでUTF-8境界を確認する。
- 本番や共有DBへ変更する前に、主キー・対象行・バックアップを確認する。
- 文字化けした値を広い条件で一括置換しない。

## まとめ

日本語が見えていることは、正しいUnicode値が保存されている証拠にならない。`HEX()`による保存バイト列の確認と、接続の`utf8mb4`指定を分けて検証すると、書き込み時の二重エンコードと読み出し時の設定不足を切り分けられる。
