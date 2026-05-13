# `npm run migrate` で発生した `ER_ACCESS_DENIED_ERROR` を `.env` の `--env-file` 読み込みで修正した

## エラーの概要

バックエンドで `npm run migrate` を実行すると、以下のエラーが発生していた。

```
Migration failed: Error: ER_ACCESS_DENIED_ERROR: Access denied for user 'root'@'localhost' (using password: NO)
```

マイグレーションスクリプトは MySQL に接続し、データベースが存在しなければ作成した上で、未適用の `.sql` ファイルを順番に実行する。接続設定（ホスト・ポート・ユーザー名）は正しいように見えたが、MySQL は接続試行を拒否し、クエリを一切実行できなかった。

エラーメッセージの中で重要なのは **「using password: NO」** という表現だ。MySQL はクライアントが空文字列をパスワードとして送信した場合にこの表現を使う。これは、環境変数が取得できていないときの典型的な症状だ。

---

## 原因

### `tsx` は `.env` を自動で読み込まない

`backend/package.json` のマイグレーションスクリプトは次のように定義されていた。

```json
"migrate": "tsx src/infrastructure/migrate.ts"
```

`tsx` は Node.js の上に構築された TypeScript 実行ツールだ。`tsx` も Node.js 本体も、`.env` ファイルを自動で読み込む仕組みを持っていない。`npm run migrate` を実行した時点で、プロセス環境には `DB_PASSWORD` が存在しない状態になっていた。

### フォールバックが無音でパスワードを空文字に変える

`src/infrastructure/migrate.ts` では、接続設定を次のように記述していた。

```ts
const connection = await createConnection({
  host: process.env.DB_HOST ?? '127.0.0.1',
  port: Number(process.env.DB_PORT ?? '3306'),
  user: process.env.DB_USERNAME ?? 'root',
  password: process.env.DB_PASSWORD ?? '',   // ← 空文字にフォールバック
  timezone: '+00:00',
  multipleStatements: true,
});
```

`?? ''` による Nullish Coalescing フォールバック自体は合理的な防御的デフォルトだが、`DB_PASSWORD` が環境に存在しない場合、**パスワードなしで接続が試みられる**ことになる。MySQL はこれを `ER_ACCESS_DENIED_ERROR` で拒否し、`using password: NO` と報告する。

### 開発サーバーで同じ問題が発生しなかった理由

`dev` スクリプトは `tsx --watch` を使っており、すでに変数がエクスポートされているシェルや、変数を注入するツールから起動された場合、`process.env` は事前に設定済みの状態になる。マイグレーションスクリプトはスタンドアロンの CLI エントリーポイントであるため、ゼロの状態から自分で環境変数を読み込む必要がある。

---

## 修正

`backend/package.json` のマイグレーションスクリプトを次のように変更した。

変更前:

```json
"migrate": "tsx src/infrastructure/migrate.ts"
```

変更後:

```json
"migrate": "tsx --env-file=.env src/infrastructure/migrate.ts"
```

`--env-file` フラグは **Node.js 22** からネイティブでサポートされており、スクリプト実行前に指定したファイルをパースして `process.env` にキーと値を注入する。`tsx` は Node.js のプロセス管理に処理を委譲するため、このフラグは透過的に受け渡される。

### `--env-file` がこのケースに最適な理由

| アプローチ | 評価 | 備考 |
|---|---|---|
| `migrate.ts` 内に `dotenv` パッケージを追加 | 動作するが依存が増える | ファイル先頭に `import 'dotenv/config'` が必要で、パッケージ追加も伴う |
| `cross-env` またはシェルの `export` | 動作するが脆弱 | シェルの状態に依存し、環境が事前設定されていない CI では壊れる |
| `--env-file`（Node.js 22 組み込み） | ✅ Node.js 22 以上ではベスト | 追加依存ゼロ。呼び出し元で明示的に指定でき、Node.js 本体にドキュメントが存在する |

このプロジェクトはすでに Node.js 22 を対象にしているため、ネイティブフラグを使うのが最もシンプルな選択だ。`migrate.ts` に環境変数読み込みのボイラープレートを追加せずに済み、`.env` への依存が `package.json` のスクリプト欄に、そのコマンドのすぐ隣に明示される――将来のメンテナーが確認するべき場所にちょうど記録されることになる。

---

## まとめ

1行の変更:

```diff
- "migrate": "tsx src/infrastructure/migrate.ts",
+ "migrate": "tsx --env-file=.env src/infrastructure/migrate.ts",
```

これにより、MySQL 接続が試みられる前に `DB_PASSWORD`（および他のデータベース認証情報）が `.env` から読み込まれるようになり、`ER_ACCESS_DENIED_ERROR` が解消された。

---

## 得られた知見

1. **「using password: NO」は空文字列が送信されたことを意味する** — MySQL でこの表現を見た場合、まず確認すべきは、そのコマンドを実行したプロセスに対象の環境変数が実際に存在するかどうかだ。

2. **CLI エントリーポイントは自分で環境変数を読み込む必要がある** — 長時間稼働するサーバーはシェルやプロセスマネージャーから環境変数を継承することが多いが、スタンドアロンスクリプト（`migrate`、`seed`、`cron` など）はゼロの状態で起動するため、`.env` を明示的に読み込まなければならない。

3. **Node.js 22 の `--env-file` は依存ゼロの解決策** — Node.js 22 以上であれば、スクリプトで `.env` を読み込むためだけに `dotenv` を導入する必要はない。`--env-file=.env`（または `tsx --env-file=.env`）を渡すだけで、ランタイムがパースを担当する。

4. **環境変数への依存を呼び出し元に明示する** — `--env-file=.env` を npm スクリプトに埋め込むことで、`package.json` のスクリプトブロックにその依存がインラインで文書化され、「このコマンドはローカルの `.env` ファイルが必要」であることが任意のエンジニアにとって明確になる。

