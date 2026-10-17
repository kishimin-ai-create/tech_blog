# 作業ブランチを切ってもupstreamがorigin/mainになることがある

## 結論

`git switch -c fix/reject-unknown-source-id origin/main`で作業ブランチを作成した際、追跡先も`origin/main`になっていた。ブランチ名だけを確認して「mainとは分離できている」と説明するのでは不十分だった。

今回のpush失敗の実際のエラー文は取得していない。そのため、この誤設定を確認した事実と、ユーザーが報告した失敗の直接原因だったという推定は分けて扱う。

## 発生した問題

API修正は専用ブランチへ2コミットした。しかし、pushできないという報告を受けて確認すると、次の状態だった。

```text
fix/reject-unknown-source-id...origin/main [ahead 2]
branch.fix/reject-unknown-source-id.remote = origin
branch.fix/reject-unknown-source-id.merge = refs/heads/main
```

ローカルmainへマージしたわけではない。問題は、作業ブランチの追跡先が共有ブランチを指していたことだった。

## 調査

Git設定とreflogを読み取りで確認した。pushの再実行や設定変更は調査中に行わなかった。

```powershell
git status --short --branch
git branch -vv
git config --get branch.fix/reject-unknown-source-id.remote
git config --get branch.fix/reject-unknown-source-id.merge
git config --get push.default
git reflog show fix/reject-unknown-source-id -3
```

`push.default`の明示設定は取得されず、reflogには`branch: Created from origin/main`が残っていた。通常の`simple`動作では、現在ブランチ名とupstreamのブランチ名が違う場合に、引数なしpushが拒否される。[Git公式のpush仕様](https://git-scm.com/docs/git-push)にもこの条件が説明されている。今回その条件に該当する設定だったが、利用者が実行したコマンドとエラー全文は未確認である。

## 修正方法と確認済みの範囲

調査時には、同名ブランチを送信先として明示し、upstreamを設定する次のコマンドを提示した。

```powershell
git push -u origin HEAD:refs/heads/fix/reject-unknown-source-id
```

これはリモートへ書き込むコマンドなので、対象のowner/repositoryとブランチを確認してから実行する。`HEAD:main`へ置き換えてはならない。

後の再確認では、追跡先が`origin/fix/reject-unknown-source-id`へ変わり、リモートにもコミット`4aa3867e82afab713c06891732865e456cccc09c`が存在した。変更した実行者と、そのときのコマンドは確認していない。提案したコマンドをこの調査で実行済みだったと扱わない。

## 再発防止の提案

ブランチ作成直後は、名前と追跡先を同時に確認する。未公開の作業ブランチでは`--no-track`を使う案もあるが、このセッションでは導入していない。

```powershell
git switch --no-track -c fix/example origin/main
git status --short --branch
git branch -vv
```

この例は将来の操作案であり、今回の実行ログではない。[Git公式のswitch仕様](https://git-scm.com/docs/git-switch)で`--no-track`の動作を確認した。push先の明示、mainのサーバー側保護、公開後のSHA確認は、それぞれ別の役割を持つ。追跡先を直しただけでサーバー保護まで検証できたとは考えない。

## 根拠と制約

根拠は2026年10月17日の`git config`、`git branch -vv`、reflog、および後続の`git ls-remote --heads origin fix/reject-unknown-source-id`の結果である。ローカルmainへのマージ・pushはこの作業では実行していない。失敗時のエラー文とサーバー側拒否ログは未取得である。
