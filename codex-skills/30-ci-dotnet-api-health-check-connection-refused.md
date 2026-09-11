# GitHub Actionsで.NET APIのhealth checkが接続拒否になったときの切り分け

## 結論

GitHub Actionsで.NET APIのビルドが成功しているのに、直後のhealth checkが`curl: (7) Failed to connect`で失敗する場合、ビルド成功とHTTP待受け開始は別に確認する必要がある。Mojicaでは、CIのバックグラウンド起動を`dotnet run --no-build`からビルド済みDLLの直接起動へ変更し、APIの起動経路を単純化した。

## 発生した問題

Pull RequestのE2Eジョブで、APIのrestoreとbuildは成功した。

```text
Build succeeded.
    0 Warning(s)
    0 Error(s)
```

しかし、`127.0.0.1:5063/health`への接続は失敗した。

```text
curl: (7) Failed to connect to 127.0.0.1 port 5063 after 0 ms: Couldn't connect to server
Error: Process completed with exit code 7.
```

## 調査

### ビルドと待受けを分けて確認する

ビルドログにはエラーがないため、コンパイルではなくAPIプロセスの起動後状態を確認した。E2Eジョブには次の環境変数が渡っていた。

```text
ASPNETCORE_URLS: http://127.0.0.1:5063
VITE_API_URL: http://127.0.0.1:5063
```

このことから、フロントエンドの接続先設定が未指定である問題とは別に、APIがhealth check時点でポートを待受けていない状態だと判断した。

### 起動コマンドを確認する

変更前は、ビルド済み成果物を使う指定でも`dotnet run`を経由していた。

```bash
nohup dotnet run --project Mojica.Api --configuration Release --no-build > ../mojica-api.log 2>&1 &
```

CIの同じstep内でhealth checkを実行しても接続できなかったため、MSBuild経由の起動処理を残さず、直前に成功したbuildの成果物を直接起動する形へ変更した。

## 解決方法

push、Pull Request、nightlyの各E2Eジョブで、次の起動方法を使用する。

```bash
dotnet build Mojica.Backend.sln --configuration Release --no-restore
nohup dotnet Mojica.Api/bin/Release/net8.0/Mojica.Api.dll > ../mojica-api.log 2>&1 &
curl --fail --retry 30 --retry-delay 1 --retry-all-errors http://127.0.0.1:5063/health
```

これにより、CIのhealth checkは実際にビルドされた`Mojica.Api.dll`を起動したプロセスに対して行われる。なお、起動プロセスが終了した詳細な例外原因は、今回のGitHub Actions出力では`mojica-api.log`の内容が表示されておらず、未確認である。

## 動作確認

修正後にローカルで実行したフロントエンド検証は成功した。

| コマンド                                           | 結果                                 |
| -------------------------------------------------- | ------------------------------------ |
| `bun run typecheck`                                | pass                                 |
| `bun run lint`                                     | pass                                 |
| `bun run test:coverage:pr`                         | pass（130 tests、Statements 96.64%） |
| `bunx prettier --check .github/workflows/ci-*.yml` | pass                                 |
| `git diff --check`                                 | pass                                 |

GitHub Actionsは修正コミット`407f116`で再実行中であり、CI上のhealth check成功はこの執筆時点では未確認である。

## 再発防止

- ビルド成功とHTTP health checkを別の検証段階として扱う。
- バックグラウンド起動するサービスは、固定時間の待機ではなくhealth endpointで待受けを確認する。
- 起動ログをArtifactとして保存し、health check失敗時にプロセス終了理由を確認できるようにする。
- フロントエンドのAPI URL設定と、APIプロセスの待受け設定を別々に確認する。

## まとめ

- `dotnet build`が成功しても、APIがHTTPポートで待受けているとは限らない。
- `curl: (7) Failed to connect`は、health check時点で対象ポートに接続できないことを示す。
- CIでは、ビルド済みDLLを直接起動し、health checkでサービスの準備完了を確認する。

確認日: 2026-09-06
