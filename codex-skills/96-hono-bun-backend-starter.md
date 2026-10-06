# HonoをBunで起動する最小バックエンド

## はじめに

API開発の入口として、既存のPython分析サービスと別にHonoの初期プロジェクトを用意した。この記事は初期起動と開発設定の確認範囲を記録する。

## 前提・環境

Windows、Bun 1.3.13、Hono 4.13.13。作業場所は `app/backend`。根拠はコミット `f865e84` と `bun.lock`。

## やってみた結果

GET / が HTTP 200 と Hello Hono! を返す初期APIを用意した。業務APIや分析サービスとの接続はまだ実装していない。

## 実装・検証

### 依存関係と起動

`package.json` のdevスクリプトは `bun run --hot src/index.ts`。依存関係を固定した状態で確認する。

```sh
bun install --frozen-lockfile
bun run dev
```

初期設定時にはインストール成功と localhost:3000 のHTTP 200を確認した。確認用プロセスは停止したため、常時起動を実現したという結果ではない。

### エントリーポイント

```ts
import { Hono } from "hono";
const app = new Hono();
app.get("/", (c) => c.text("Hello Hono!"));
export default app;
```

これは登録済みルートを示す最小例である。現在のソースではコールバックにブロックを使っている。

## つまずいたところ

初期起動で確認した障害はない。後続のBiome設定とテスト検証は別の記事に分けた。

## 学んだこと

FACT: 起動成功を確認した範囲は初期ルートのみ。INFERENCE: APIを追加するための入口はできたが、既存サービスとの統合完了を示す証拠にはならない。

## まとめ

初期起動の確認と業務機能の完成を分けて記録する。型チェック専用コマンドやカバレッジ取得はこの時点では設定していない。

