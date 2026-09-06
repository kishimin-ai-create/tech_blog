# Bunを残したままHono APIをCloudflare Workersへ移す設計

## 結論

Bunで開発するHono APIをCloudflare Workersへ移すために、Bunを捨てる必要はない。今回の構成では、Bunを依存管理・テスト・ローカルサーバー・DBマイグレーションに残し、本番のHTTP実行境界だけをWorkersへ追加した。

データベースはSupabase PostgreSQLを維持し、WorkerからCloudflare Hyperdriveを通して接続する。DBマイグレーションはWorkerの起動時やAPIリクエスト中に実行せず、デプロイ前の独立した工程へ分離した。

```text
Next.js server-side API proxy
  -> Cloudflare Worker (Hono)
  -> Hyperdrive
  -> Supabase PostgreSQL
```

## 背景

対象backendはHono、TypeScript、Drizzle ORM、`pg`で構成され、Bun用エントリーポイントがポートを持つ常駐プロセスと起動時マイグレーションを提供していた。一方、Cloudflare Workersではサーバーを起動せず、`fetch`ハンドラーを公開する。この実行モデルの違いに合わせて、サーバーとDB接続のライフサイクルを分離する必要があった。

## 解決したい課題

- Bunで動く既存の開発・テスト環境を壊さない
- HonoのController、Service、Repository境界を維持する
- Supabase PostgreSQLをD1へ置き換えない
- 複数のWorker isolateからマイグレーションを開始しない
- Supabaseの接続情報とJWT SecretをGitへ保存しない

## 前提・制約

検証時の主なバージョンは次のとおりだった。

| 対象 | バージョン |
| --- | --- |
| Bun | 1.3.13 |
| Wrangler | 4.129.0 |
| Hono | 4.12.23 |
| Drizzle ORM | 0.45.2 |
| `pg` | 8.22.0 |

CloudflareのSupabase接続ガイドでは、HyperdriveへSupabaseのDirect connectionを登録し、Workerでは`pg`などからHyperdriveの接続文字列を使う構成が案内されている。Supabase側のPoolerへHyperdriveを重ねる構成にはしなかった。

## 検討した選択肢

### Option A: Bunサーバーをコンテナへデプロイする

既存のポート、接続プール、起動時マイグレーションを維持できるが、backendをWorkersで運用する今回の目的を満たさない。

### Option B: PostgreSQLをD1へ移行する

WorkerからCloudflare bindingだけでDBへアクセスできる。一方で、PostgreSQLを使うデータ要件、Drizzleスキーマ、既存データまで変更対象になる。

### Option C: Bunを残し、WorkersからHyperdrive経由でSupabaseへ接続する

Hono、Drizzle、PostgreSQLの既存境界を維持できる。ただし、Bun用とWorkers用のエントリーポイント、型設定、Hyperdrive、Secretsを別々に管理する必要がある。

## 評価軸

| 評価軸 | コンテナ | D1 | Workers＋Hyperdrive |
| --- | --- | --- | --- |
| Workersで実行 | 不可 | 可 | 可 |
| PostgreSQL要件を維持 | 可 | 不可 | 可 |
| 既存Repositoryを維持 | 可 | 大幅変更 | 可 |
| Bun開発環境を維持 | 可 | 移行内容次第 | 可 |
| 新しい運用設定 | コンテナ設定 | D1移行 | Hyperdrive、Secrets |

## 最終判断

Option Cを採用した。Bun用の`src/index.ts`は残し、Workers専用の`src/worker.ts`を追加した。両者は同じHonoアプリケーションを利用する。

```ts
export default {
  fetch(request: Request, env: Env): Promise<Response> {
    return workerHandler.fetch(request, {
      databaseUrl: env.HYPERDRIVE.connectionString,
      jwtSecret: env.JWT_SECRET,
    });
  },
} satisfies ExportedHandler<Env>;
```

DB接続はリクエスト単位で作成し、Honoの処理後に必ず閉じる。

```ts
const requestApp = await dependencies.createRequestApp(config);
try {
  return await requestApp.app.fetch(request);
} finally {
  await requestApp.close();
}
```

## なぜこの選択をしたか

実行基盤の変更とデータ契約の変更を分離できるからである。Workersへの移行とD1への移行を同時に行うと、実行環境、SQL方言、スキーマ、データ移行を一度に扱うことになる。Hyperdriveを境界に置けば、既存の`pg`とDrizzle Repositoryを維持したまま、接続ライフサイクルだけをWorkersへ合わせられる。

マイグレーションもHTTP境界から外した。Worker isolateの生成を「サーバーが一度だけ起動した」と見なせないため、SupabaseのDirect connectionを使う`bun run db:migrate`をデプロイ前に独立して実行する。

## 実装後の結果

| 検証 | 結果 |
| --- | --- |
| Bun・Workers TypeScript型チェック | 成功 |
| ESLint | エラー0、既存warning 10件 |
| Bunテスト | 131件成功 |
| Function coverage | 85.85% |
| Line coverage | 89.28% |
| `wrangler deploy --dry-run` | 成功 |
| Worker bundle | 1,162.64 KiB、gzip 197.17 KiB |
| ローカルstartup profile | Active 85.7 ms |

本番Hyperdrive IDとSecretsは未設定だったため、Cloudflareへの実デプロイとSupabaseへの本番疎通は未実施である。

## トレードオフ・今後の懸念

- Hyperdrive IDと`JWT_SECRET`を本番環境へ設定する必要がある
- デプロイ前にSupabaseへマイグレーションを適用する必要がある
- 同期版`scrypt`のCPU時間は実Worker上で計測する必要がある
- Frontend proxyへWorker URLを`BACKEND_URL`として設定する必要がある

## まとめ

BunはWorkersと競合するものではなく、開発・テスト・マイグレーションの道具として残せる。本番HTTP境界だけをWorkersへ分離し、Supabase PostgreSQLとの間にHyperdriveを置くことで、既存のHono APIとデータ契約を維持できた。

ただし、dry-runの成功は本番疎通の証明ではない。Hyperdrive、Secrets、マイグレーション、認証の人間レビューと本番スモークテストが残っている。

## 参考資料

- [Cloudflare Workers: Hono](https://developers.cloudflare.com/workers/framework-guides/web-apps/more-web-frameworks/hono/)
- [Cloudflare Hyperdrive: Supabase](https://developers.cloudflare.com/hyperdrive/examples/connect-to-postgres/postgres-database-providers/supabase/)
- [Cloudflare Workers Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
- 確認日: 2026年9月6日
