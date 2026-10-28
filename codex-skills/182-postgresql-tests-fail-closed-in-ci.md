# PostgreSQLテストの未実行をCIで成功扱いにしない

ローカル開発ではDBを用意していない場合もある。一方、CIで接続先の設定が漏れたときにDBテストを静かにスキップすると、成功表示だけが残る。

profile-hubでは `TEST_DATABASE_URL` がないローカル環境ではDBテストを省略できるようにし、CIでは `REQUIRE_TEST_DATABASE=1` を設定した。必須モードでは接続先がなければ失敗する。GitHub ActionsはPostgreSQL 17のサービスを起動し、マイグレーション後にDBテストを実行する。テストではトランザクションのロールバックで変更を隔離する。

[PR #64](https://github.com/kishimin-ai-create/profile-hub/pull/64) のPostgreSQL integrationジョブの成功を確認した。これはそのPRの対象コミットについての結果であり、後続PRや本番環境の結果ではない。PRやNightlyでの別のCI設定は後続の変更として扱い、実際に走ったイベントを個別に確認する必要がある。

ローカルで便利な省略と、CIで必要な失敗を明示的に分けることで、未実行を成功と取り違えにくくなる。
