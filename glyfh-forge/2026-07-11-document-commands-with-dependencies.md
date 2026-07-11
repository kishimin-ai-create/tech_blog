# READMEのコマンドは依存関係とセットにする

README にコマンドを書くなら、そのコマンドがインストール手順だけで動く必要がある。

今回の例はこれ。

```bash
uvicorn app.main:app --reload
```

README には API サーバーの起動方法として `uvicorn` を書いた。
でも `requirements.txt` に `uvicorn` が入っていなければ、初めて触る人はそこで止まる。

修正は小さい。

```text
uvicorn==0.30.1
```

これを依存に追加した。

README は説明書であり、手順書でもある。
書いたコマンドがそのまま動くことは、かなり大事。
