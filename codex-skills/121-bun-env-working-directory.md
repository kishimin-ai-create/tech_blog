# Bunで環境変数が未定義になったら起動ディレクトリを確認する

## 結論

今回のローカル検証では、`app/.env`の変数を取得できるかどうかがBunの起動ディレクトリによって変わった。`hono/adapter`の`env(c)`はBun環境で`process.env`を参照するため、先にランタイムが環境変数を読み込んでいる必要がある。

ただし、ユーザーが実際に使った起動コマンドは確認できていない。報告された通信失敗の原因を、この観測だけで断定はしていない。

## 発生した問題

画像APIのURLをlocalhostで直接指定すると呼べるが、環境変数にすると呼べないという相談があった。対象は`app/backend/src/routes/images/images.ts`、設定は`app/.env`にあった。

検証環境はWindows、Bun 1.3.13、Hono 4.13.13。秘密値は出力せず、変数の有無とURLとしての妥当性だけを確認した。

## 調査

次のコードを、リポジトリ直下と`app`直下でそれぞれBunから実行した。

```ts
const value = process.env.KOTOGUMO_FAST_API_URL;
console.log({
  defined: value !== undefined,
  validUrl: !!value && URL.canParse(value),
});
```

| 実行場所 | 定義 |
| --- | --- |
| リポジトリ直下 | 未定義 |
| app直下 | 定義あり。有効なHTTP URLであることも確認 |

インストール済みの`hono/dist/helper/adapter/index.js`では、Bunに対する取得処理はグローバルの`process.env`を返していた。

## 原因と解決方法

変数が未定義のままテンプレート文字列へ入ると、fetch先は`undefined/images/1`になる。これはコードから導ける結果であり、元のサーバープロセスの実測URLではない。

読み込みを確認できた起動方法は次のとおり。

```powershell
cd app
bun run dev
```

調査時のdevスクリプトは`bun run --hot backend/src/index.ts`だった。

## 動作確認

`app`直下で画像ルートをモック付きで呼び出し、設定したURLがfetch先に使われることとHTTP 200を確認した。実APIとの通信成功は未確認である。

## 再発防止

提案として、起動手順に作業ディレクトリを明記し、必須環境変数を起動時に検証すると切り分けが早くなる。この検証処理は今回追加していない。

## まとめ

変数名が正しくても、起動したプロセスに値が入っているとは限らない。値を表示する前に、有無・形式・実行場所を確かめる。

## 根拠

2026年10月17〜18日のblog-wordcloud調査。対象ファイルは`app/.env`、`app/package.json`、`app/backend/src/routes/images/images.ts`。環境変数の取得実装はインストール済みのHonoを確認した。
