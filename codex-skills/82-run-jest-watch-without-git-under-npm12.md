# Git管理外でJestをwatchする：npm 12では`npx jest --watchAll`を使う

## 結論

この作業環境では、npm 12.0.2経由でJestへ引数を渡す検証が`EUNKNOWNCONFIG`で失敗し、Jestの`--watch`もGit管理外のフォルダーでは起動できなかった。プロジェクトにJestがインストール済みなら、`npx jest --watchAll --runInBand`でGitを必要としない全ファイル監視を起動できる。

## 発生した問題

テストスクリプトは次のとおりだった。

```json
{
  "scripts": {
    "test": "jest --runInBand"
  }
}
```

一般的なnpm scriptの引数転送を試すと、この環境ではnpmがJestの引数を受け付けず、`EUNKNOWNCONFIG`と`Unknown cli flags`を返した。

最初に案内した`npm test -- --watch`は、動作確認をせずに提示した誤った案内だった。実際に確認したのは上記の`npm run test -- --watch --listTests`であり、npmが未知のCLI設定として拒否した。

## 調査

環境で確認したバージョンはnpm 12.0.2とNode.js 26.7.0だった。

```powershell
npm run test -- --watch --listTests
```

このコマンドはJestを起動せず、npmが`--watch`と`--listTests`を未知のCLI設定として拒否した。

Jestを直接実行するため`npx`へ切り替えた。

```powershell
npx jest --watch --runInBand --listTests
```

Jestは起動したが、次のメッセージで終了した。

```text
--watch is not supported without git/hg, please use --watchAll
```

このワークスペースにはGitまたはMercurialのリポジトリがなかった。Jestの`--watch`は変更ファイルをバージョン管理から絞るため、バージョン管理の作業ツリーが必要だった。

## 解決方法

`--watch`を`--watchAll`へ変える。

```powershell
npx jest --watchAll --runInBand
```

`--watchAll`はバージョン管理状態に依存せず、すべてのテストファイルを監視する。確認時には`--listTests`を加え、2つのJestテストファイルが検出されることと、プロセスが監視状態を保つことを確認した。

## 動作確認

```powershell
npx jest --watchAll --runInBand --listTests
```

2つのテストファイルが出力され、監視プロセスは継続した。確認後にCtrl+Cで終了した。

通常起動時は`--listTests`を外す。対話ターミナルでJestを使う場合は、Git管理内なら`--watch`、Git管理外なら`--watchAll`を選ぶ。

## 事実・判断・未確認事項

- FACT: npm 12.0.2で`npm run test -- --watch --listTests`は`EUNKNOWNCONFIG`になった。
- FACT: Git管理外でJestの`--watch`は拒否され、`--watchAll`はテストファイルを列挙して監視を継続した。
- 未確認: npmやJestの他バージョンで同じ引数処理になるかは検証していない。
- 未確認: ファイルを編集した後にJestが再実行されるところまでは、この確認では実演していない。

## まとめ

watch起動の問題はnpmの引数解釈とJestのバージョン管理要件に分けて調べる。npm script経由で引数を渡せない環境では`npx jest`を使い、Gitがない場合は`--watchAll`を選ぶ。

確認日: 2026年9月28日
