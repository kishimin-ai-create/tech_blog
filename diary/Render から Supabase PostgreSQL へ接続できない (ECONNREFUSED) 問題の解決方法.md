# Render から Supabase PostgreSQL へ接続できない (`ECONNREFUSED`) 問題の解決方法

## はじめに

Render からは IPv6 経由で正常に接続できず、結果として接続拒否が発生していました。

## 対象読者

- Render で Hono / Node.js アプリを運用している方
- Supabase PostgreSQL を利用している方
- デプロイ後に DB 接続エラーが発生している方

## エラー概要

Render へデプロイしたアプリで、以下のようなエラーが発生しました。

```txt
connect ECONNREFUSED 2406:da18:e5c:b700:2251:77fa:a75b:934b:5432
```

また、アプリケーションログには次のようなメッセージが出力されていました。

```txt
migration error message connect ECONNREFUSED ...
database migrations failed
request blocked while database migrations are not ready
```

## 原因

当初は Drizzle Migration や PostgreSQL のテーブル定義に問題があると考えていました。

しかしログを詳細に確認したところ、実際には Migration 実行前のデータベース接続で失敗していました。

```txt
ECONNREFUSED
```

は PostgreSQL サーバーが接続を拒否していることを意味します。

今回のケースでは、Supabase の Direct Connection を利用していました。

```txt
db.xxxxx.supabase.co:5432
```

この接続先は環境によって IPv6 接続となる場合があります。

Render からは IPv6 経由で正常に接続できず、結果として接続拒否が発生していました。

## 解決方法

Supabase の Connection Pooler を利用します。

Supabase Dashboard の以下から接続情報を取得できます。

```txt
Project Settings
  └ Database
      └ Connection String
```

その中の Pooler 接続文字列を利用します。

例:

```txt
postgresql://postgres.xxxxx:password@aws-xxx.pooler.supabase.com:6543/postgres
```

重要なのは以下の 2 点です。

```txt
host: *.pooler.supabase.com
port: 6543
```

## Render の設定変更

Render の Environment Variables に設定している

```txt
DATABASE_URL
```

を Pooler 用の接続文字列へ変更します。

変更前:

```txt
postgresql://user:password@db.xxxxx.supabase.co:5432/postgres
```

変更後:

```txt
postgresql://user:password@aws-xxx.pooler.supabase.com:6543/postgres
```

設定後に再デプロイします。

## 注意点

Supabase では用途によって接続方法が異なります。

### Direct Connection

```txt
db.xxxxx.supabase.co:5432
```

- 開発環境向け
- 接続数が増えると不利
- 環境によって IPv6 問題が発生する場合がある

### Pooler Connection

```txt
*.pooler.supabase.com:6543
```

- 本番環境向け
- 接続プーリング対応
- Render やサーバーレス環境との相性が良い

## まとめ

今回の原因は Drizzle Migration ではなく、Render から Supabase への PostgreSQL 接続設定でした。

Render で Supabase を利用する場合は、Direct Connection ではなく Pooler
Connection を使用することで接続エラーを回避できます。

特に本番環境では、接続安定性と接続数管理の観点からも Pooler の利用を推奨します。
