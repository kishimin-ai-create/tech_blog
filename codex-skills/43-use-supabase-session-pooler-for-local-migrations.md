# Supabase Direct connectionでENOTFOUNDになったときのmigration経路

## 結論

Cloudflare Hyperdriveとローカルのmigrationでは、同じSupabase接続経路を使う必要はない。今回のWindows環境では、HyperdriveにはSupabase Direct connectionを設定し、ローカルのDrizzle migrationにはSession poolerを使うことで接続できた。

Direct connectionを使ったローカルmigrationは`getaddrinfo ENOTFOUND`で失敗した。対象ホストがAAAAレコードだけを返すことを確認したため、IPv4で到達できるSession poolerへmigration経路を切り替えた。

## 発生した問題

### 症状

`DATABASE_URL`へSupabase Direct connectionを設定してmigrationを実行すると、SQLの実行前にDNS解決で失敗した。

```text
DNSException: getaddrinfo ENOTFOUND
syscall: getaddrinfo
code: ENOTFOUND
```

### 発生条件

- Environment: Windows、PowerShell
- Bun: 1.3.13
- Migration: Drizzle Kit / Drizzle runtime migrator
- Database: Supabase PostgreSQL
- Production connection: Cloudflare Hyperdrive

### 影響

Hyperdriveの作成には成功したが、デプロイ前にローカルからSupabaseへmigrationを適用できなかった。

## 調査

### 接続文字列の入力を疑った

最初に、クリップボードの値を`System.Uri`で解析し、接続文字列そのものを表示せずscheme、host、portを検査した。入力が空だったケースと、接続文字列以外の複数行テキストがコピーされていたケースを切り分けた。

有効なURIをコピーした後もDirect connectionでは同じエラーになったため、DNSレコードを確認した。

```powershell
Resolve-DnsName -Name db.<PROJECT_REF>.supabase.co
```

対象ホストはAAAAレコードだけを返した。一方、Session poolerのホストはWindows環境から名前解決できた。

### Hyperdrive側の接続を確認した

Cloudflare側ではDirect connectionを使ったHyperdrive構成を作成できた。したがって、Supabaseの認証情報全体が誤っているのではなく、ローカルWindowsからDirect connectionホストへ到達する経路が問題だと判断した。

## 原因

Supabase Direct connectionのホストが今回の環境ではIPv6アドレスだけを返し、Windows上のBunと`pg`から名前解決・接続できなかったことが原因である。

`ENOTFOUND`はmigration SQLやDrizzle schemaのエラーではない。PostgreSQLへ接続する前のDNS段階で停止していた。

## 解決方法

接続経路を用途別に分けた。

| 用途 | 接続経路 |
| --- | --- |
| Cloudflare Worker | Hyperdrive → Supabase Direct connection |
| Windows上のmigration | Supabase Session pooler、port 5432 |

Session poolerのURIはSupabase Dashboardから取得し、ファイルへ保存せず一時的に環境変数へ設定した。

```powershell
$env:DATABASE_URL = $sessionPoolerUrl

try {
    bun run db:migrate:runtime
}
finally {
    Remove-Item Env:DATABASE_URL
}
```

HyperdriveへPoolerの接続文字列を登録し直す変更は行っていない。Hyperdrive自身が接続をpoolするため、Cloudflare公式のSupabase接続手順どおりDirect connectionを維持した。

## 動作確認

```text
database connection succeeded
database migrations completed successfully
```

続けてWorkersをデプロイし、次を確認した。

| 確認 | 結果 |
| --- | --- |
| Worker deployment | 成功 |
| `GET /health` | 200 |
| `GET /api/diaries` | 200、既存データを取得 |

この結果から、Session pooler経由で適用したschemaを、HyperdriveのDirect connection経由でも利用できることを確認した。

## 再発防止

- `ENOTFOUND`ではSQLより先に、解析されたhostとDNSレコードを確認する
- 接続文字列をログへ出さず、scheme、host、portだけを検証する
- Hyperdriveとローカルmigrationの接続経路を同一にすることを前提にしない
- migration後はDB接続成功だけでなく、Worker経由のAPI応答も確認する

## まとめ

- 症状: Supabase Direct connectionでmigrationすると`getaddrinfo ENOTFOUND`になった
- 原因: Direct connectionホストがAAAAレコードだけを返し、今回のWindows環境から接続できなかった
- 解決: migrationだけSession poolerへ切り替えた
- 教訓: 本番ランタイムとデプロイ作業の接続経路は、実行環境のネットワーク条件に合わせて分離できる

## 参考資料

- [Cloudflare HyperdriveからSupabaseへ接続する](https://developers.cloudflare.com/hyperdrive/examples/connect-to-postgres/postgres-database-providers/supabase/)
- [Supabase PostgreSQLへの接続方法](https://supabase.com/docs/guides/database/connecting-to-postgres)
- 確認日: 2026年9月6日
