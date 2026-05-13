# Vitest 4でフロントエンドとバックエンドのカバレッジレポートをActionsに統合した

## 対象読者

- React/Vite フロントエンドと Node.js バックエンドを同じリポジトリで管理している人
- Vitest の coverage を GitHub Actions で artifact として残したい人
- フロントエンド、バックエンド、統合ダッシュボードをひとつの coverage フローにまとめたい人

## この記事の範囲

この記事では、`frontend` と `backend` の coverage を生成し、GitHub Actions でレポートを artifact として保存する構成について扱います。

一方で、Codecov や Coveralls のような外部サービス連携、coverage trend の永続管理、README badge の自動更新までは扱いません。

## 背景

このリポジトリでは、フロントエンドは React/Vite/Vitest、バックエンドは Hono/Vitest で構成されています。

coverage reporting の仕様は `docs/spec/features/test-coverage-reporting.md` に整理され、主な要件は次の通りでした。

- frontend の coverage を `frontend/coverage/` に出力する
- backend の unit coverage を `backend/coverage/unit/` に出力する
- backend の integration coverage を `backend/coverage/integration/` に出力する
- line、branch、function、statement のしきい値を設定する
- `docs/coverage/index.html` に中央ダッシュボードを生成する
- GitHub Actions で coverage report を artifact として保存する

単に `npm run coverage` を追加するだけでは、フロントエンドとバックエンドの出力場所、CI の失敗時の artifact 回収、統合ダッシュボード生成までを扱えません。そこで root scripts、Vitest config、GitHub Actions、集約スクリプトをまとめて整備しました。

## 実装内容

### root から coverage を一括実行する

ルートの `package.json` に coverage 用のスクリプトを追加しました。

```json
{
  "scripts": {
    "coverage": "npm run coverage:frontend && npm run coverage:backend && npm run coverage:report",
    "coverage:frontend": "cd frontend && npm run coverage",
    "coverage:backend": "cd backend && npm run coverage",
    "coverage:report": "node scripts/generate-coverage-report.js"
  }
}
```

これにより、個別実行と集約実行を分けられます。

- frontend だけ確認したい場合は `npm run coverage:frontend`
- backend だけ確認したい場合は `npm run coverage:backend`
- 最後に中央ダッシュボードだけ作り直したい場合は `npm run coverage:report`

### Vitest 4 の coverage 設定に合わせる

coverage のしきい値は、frontend の `vite.config.ts`、backend の `vitest.unit.config.ts` と `vitest.integration.config.ts` に設定しました。

Vitest 4 では coverage threshold は `coverage.thresholds` 配下に置きます。

```ts
coverage: {
  provider: 'v8',
  reporter: ['html', 'json'],
  reportsDirectory: './coverage',
  thresholds: {
    lines: 80,
    branches: 75,
    functions: 80,
    statements: 80,
  },
}
```

backend は unit と integration でレポートの出力先を分けています。

- `backend/coverage/unit`
- `backend/coverage/integration`

この分離により、unit test と integration test の coverage を同じ backend coverage として混ぜずに確認できます。

### 中央ダッシュボードを生成する

`scripts/generate-coverage-report.js` を追加し、次の JSON coverage を読み取るようにしました。

- `frontend/coverage/coverage-final.json`
- `backend/coverage/unit/coverage-final.json`
- `backend/coverage/integration/coverage-final.json`

読み取った metrics から `docs/coverage/index.html` を生成します。

このスクリプトは coverage ファイルが存在しない場合でも `null` として扱い、0% の metrics に落とす実装になっています。そのため、CI 上で一部 coverage が失敗しても、生成可能な範囲で dashboard artifact を残せます。

### GitHub Actions で artifact を保存する

`.github/workflows/test-coverage.yml` を追加し、次を実行するようにしました。

1. frontend dependencies を `npm ci`
2. backend dependencies を `npm ci`
3. frontend coverage を実行
4. backend unit coverage を実行
5. backend integration coverage を実行
6. 中央 dashboard を生成
7. 3種類の artifact を upload

artifact は次の名前で保存されます。

- `frontend-coverage-report`
- `backend-coverage-report`
- `combined-coverage-dashboard`

coverage 実行 step には `continue-on-error: true` を設定し、最後の step で失敗判定を戻しています。

```yaml
- name: Fail when coverage failed
  if: steps.frontend_coverage.outcome == 'failure' || steps.backend_unit_coverage.outcome == 'failure' || steps.backend_integration_coverage.outcome == 'failure'
  run: exit 1
```

この構成にすると、coverage が落ちても artifact upload は実行されます。失敗原因を確認するための HTML/JSON レポートを残せる点が重要です。

## テストの扱い

coverage reporting の設定を確認するテストは、frontend と backend にそれぞれ追加しました。

- `frontend/src/test/coverage.spec.ts`
- `backend/src/tests/coverage-integration.spec.ts`

ここでは `npm run coverage` をテスト中に再帰実行せず、scripts、config、dashboard generator の存在と設定を確認する軽量なテストにしています。

通常の `npm test` に coverage generation を混ぜると、テスト実行中にさらに Vitest が起動し、タイムアウトや一時ファイル競合の原因になります。coverage の実生成は専用 command と GitHub Actions に寄せ、通常テストでは設定が壊れていないことを確認する役割にしました。

## 注意点

`docs/spec/features/test-coverage-reporting.md` には badge 更新や history tracking も将来要件として書かれています。ただし、今回の実装では dashboard generation と artifact upload が中心です。

また、`README.md` には coverage dashboard への導線が追加されていますが、実際の badge percentage を自動更新する仕組みまでは実装していません。

## まとめ

今回の変更では、frontend、backend unit、backend integration の coverage をそれぞれ生成し、中央 dashboard と GitHub Actions artifact にまとめる流れを作りました。

ポイントは次の3つです。

- Vitest 4 に合わせて `coverage.thresholds` を使う
- backend は unit/integration の coverage 出力先を分ける
- CI では coverage 失敗時も artifact を残し、最後に失敗判定を戻す

coverage は数値そのものよりも、失敗時に原因を追える形で残ることが重要です。今回の workflow はその確認経路を整えるための土台になっています。
