# `npm run dev` が `.env` を読み込まない問題を `--env-file-if-exists` で修正した

## 対象読者

- Node.js + tsx でバックエンド開発サーバーを起動している開発者
- `.env` ファイルでローカルの DB 接続情報を管理しているチーム
- `--env-file` と `--env-file-if-exists` の違いを知りたいエンジニア

---

## 問題の背景

このプロジェクトでは MySQL の接続情報（`DB_HOST`, `DB_USER`, `DB_PASSWORD` など）を `.env` ファイルで管理している。`npm run migrate` スクリプトには既に `--env-file-if-exists=.env` が付いており、`.env` から環境変数を読み込む仕組みになっていた。

ところが `npm run dev` には同等のオプションが付いていなかった。

```json
// 修正前
"dev": "tsx --watch src/server.ts"
```

tsx は Node.js のフラグをそのまま透過するため、`--env-file` 系のオプションを付けない限り `.env` ファイルは**一切読み込まれない**。シェルの環境変数に `DB_*` を設定していない開発者がローカルで `npm run dev` を実行すると、次のような MySQL 接続エラーが発生する。

```
Error: Access denied for user ''@'localhost' (using password: NO)
```

これはコードレビュー（P1 指摘）で発覚し、修正対応した。

---

## 原因

Node.js には `.env` ファイルを自動で探索して読み込む仕組みがない。`dotenv` パッケージを使うか、起動時フラグで明示しない限り `process.env` に `.env` の内容は反映されない。

tsx（TypeScript 実行エンジン）は Node.js のフラグをそのまま利用するため、`--env-file` 系オプションを渡すことで `.env` 読み込みが可能になる。

### `--env-file` vs `--env-file-if-exists`

Node.js には似た名前のフラグが2つある。

| フラグ | `.env` が存在する場合 | `.env` が存在しない場合 |
|---|---|---|
| `--env-file=.env` | ファイルを読み込む | `ERR_INVALID_ARG_VALUE` をスローして終了 |
| `--env-file-if-exists=.env` | ファイルを読み込む | エラーなくスキップ |

`--env-file` を使うと、`.env` を持たない CI 環境やシェルの環境変数で設定している開発者が`npm run dev`を実行できなくなる。`--env-file-if-exists` は **ファイルが存在しないケースに寛容**であり、`.env` がある場合だけ読み込んでくれる。

> `--env-file-if-exists` は Node.js 22.10 で追加された。このプロジェクトは v22.20.0 を使用しているため利用可能。

---

## 解決策

`backend/package.json` の `dev` スクリプトに `--env-file-if-exists=.env` を追加した。

```json
// 修正後
"dev": "tsx --env-file-if-exists=.env --watch src/server.ts"
```

`migrate` スクリプトはすでに同フラグを使っていた。

```json
"migrate": "tsx --env-file-if-exists=.env src/infrastructure/migrate.ts"
```

これで `dev` と `migrate` の両スクリプトが同じ方針で `.env` を読み込むようになり、一貫性が保たれた。

---

## 動作の整理

修正後の挙動は次のとおり。

| 状況 | `npm run dev` の挙動 |
|---|---|
| `.env` ファイルが存在する | ファイルから `DB_*` などを読み込んで起動 |
| `.env` ファイルが存在しない | エラーなし。シェルの環境変数を使用 |
| CI（`.env` なし、シェル変数あり） | 問題なく動作 |

---

## 注意点

### Node.js バージョンの確認

`--env-file-if-exists` は Node.js **22.10 以降**でのみ使用できる。それより古いバージョンでは認識されずエラーになる。

```bash
node --version   # v22.10.0 以上であることを確認
```

### tsx との組み合わせ

tsx はフラグを Node.js にそのまま渡す。したがって Node.js 側に`--env-file-if-exists`のサポートがあれば、tsx 自体のバージョンに関わらず動作する。

### `.env` の gitignore 確認

`.env` はリポジトリに含めないのが原則。`.gitignore` に `.env` が記載されていることを確認すること。

---

## まとめ

- `npm run dev` に `--env-file-if-exists=.env` を追加することで、`.env` ファイルが存在する場合だけ自動的に環境変数を読み込むようになった
- `--env-file` はファイル不在時にエラーを出すが、`--env-file-if-exists` はスキップするため CI やファイルなし環境に対して安全
- `dev` と `migrate` の両スクリプトが同じフラグを使うようになり、`.env` 読み込みの挙動が一貫した
- コミット: `fix: load .env in npm run dev via --env-file-if-exists` (`5821e81`)
