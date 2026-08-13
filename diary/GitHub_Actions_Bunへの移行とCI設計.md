# GitHub Actions を npm から Bun に移行した話：壊れていた CI を直しながら設計し直す

## はじめに

本記事ではその問題点を整理し、Bun ベースのワークフローへ移行した過程と、最終的な CI 設計を解説する。

## 対象読者

- Bun を使ったプロジェクトの GitHub Actions が壊れていて直したい人
- モノレポの CI を設計したいエンジニア
- `actions/checkout@v5` のような「存在しないバージョン」による失敗を経験した人
- CI の段階（push / PR / ナイトリー）でテストの範囲を変えたい人

## 概要

フロントエンドの設定が整ったタイミングで GitHub Actions のワークフローを整備した。  
既存のワークフローは npm/Node.js ベースで書かれており、実際には動かない設定が多数含まれていた。  
本記事ではその問題点を整理し、Bun ベースのワークフローへ移行した過程と、最終的な CI 設計を解説する。

---

## 問題の全体像

移行前のワークフローに含まれていた問題を列挙する。

### 1. `actions/checkout@v5` は存在しない

```yaml
# ❌ 移行前
- uses: actions/checkout@v5
```

`actions/checkout` の最新安定版は `v4` であり、`v5` は存在しない。このままでは CI が起動直後に失敗する。

```yaml
# ✅ 修正後
- uses: actions/checkout@v4
```

### 2. npm/Node.js で動かそうとしていた

バックエンドは Bun ランタイムで動作し、`package.json` にも `bun install` を前提とした設定になっているにもかかわらず、ワークフローでは `actions/setup-node` + `npm ci` を使っていた。

```yaml
# ❌ 移行前
- uses: actions/setup-node@v4
  with:
    node-version: "20"
- run: npm ci
```

```yaml
# ✅ 修正後
- uses: oven-sh/setup-bun@v1
  with:
    bun-version: latest
- run: bun install --frozen-lockfile
```

`bun install --frozen-lockfile` は `npm ci` に相当し、`bun.lock` の内容と完全に一致しない場合はエラーになる。

### 3. `pwsh --version` は Windows 専用コマンド

```yaml
# ❌ 移行前（ubuntu-latest で実行される）
- run: pwsh --version
```

`pwsh` は PowerShell のコマンドであり、`ubuntu-latest` ランナーでは使えない。Bun のバージョン確認に置き換えた。

```yaml
# ✅ 修正後
- run: bun --version
```

### 4. 存在しないスクリプトを呼んでいた

実際の `package.json` に定義されていないスクリプト名を CI で呼んでいた。

| ❌ ワークフロー上のスクリプト | ✅ 実際に定義されているスクリプト |
|---|---|
| `test:e2e:medium` | `e2e` |
| `test:e2e:large` | `e2e` |
| `coverage:unit` | `test:coverage` |
| `coverage:integration` | `test:coverage` |
| `coverage:report` | （不要、coverage 実行時に自動生成）|

### 5. DB 環境変数の名前が違った

`test-coverage.yml` で MySQL サービスに接続するときの環境変数が、バックエンドの `.env.example` で定義された変数名と一致していなかった。

```yaml
# ❌ 移行前
DB_DATABASE: TDDTodoAppDB
DB_USERNAME: root
```

```yaml
# ✅ 修正後
DB_NAME: diary_db
DB_USER: root
```

### 6. バックエンドのビルド・マイグレーションステップが不要だった

移行前のワークフローには `bun run build` や `bun run migrate` のステップが含まれていたが、Bun は TypeScript を直接実行できるためビルドステップは不要。マイグレーションは CI の各テストジョブで都度実行するものではない。

---

## 修正後のワークフロー設計

ブランチイベントに応じて 3 段階のテスト実行戦略を取る。

```
main/develop への push  →  ci.yml         (small テストのみ)
Pull Request 作成/更新  →  ci-pr.yml      (small + medium + E2E)
毎朝 3:00 JST          →  ci-nightly.yml  (全テスト + VRT)
```

`test-coverage.yml` はカバレッジ計測専用で、push / PR どちらでもトリガーされる。  
`copilot-setup-steps.yml` は GitHub Copilot エージェントが使う環境セットアップ専用。

### ci.yml — main/develop ブランチへの push

```yaml
on:
  push:
    branches: [main, develop]
```

バックエンド・フロントエンドそれぞれのジョブで lint → typecheck → small テスト → build を実行する。最も高頻度で動くワークフローのため、small テスト（ユニットテスト）に限定してフィードバックを速くしている。

### ci-pr.yml — Pull Request

```yaml
on:
  pull_request:
```

small + medium テストを実行し、さらにフロントエンドでは Storybook ビルド・E2E テストも実行する。テスト結果（`test-result.json`）と Playwright レポートをアーティファクトとしてアップロードするため、失敗時の調査がしやすい。

```yaml
- name: Upload test results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: frontend/test-result.json
    if-no-files-found: ignore
    retention-days: 7

- name: Upload Playwright report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: playwright-report
    path: frontend/playwright-report/
    retention-days: 7
```

`if: always()` でテスト失敗時にもアーティファクトがアップロードされる点が重要。

### ci-nightly.yml — ナイトリー（JST 03:00）

```yaml
on:
  schedule:
    - cron: '0 18 * * *'  # UTC 18:00 = JST 03:00
  workflow_dispatch:
```

全テスト（small + medium + large）を実行した後、VRT を実行する。

```yaml
- name: Capture VRT screenshots
  run: bunx storycap --serverCmd "bunx serve storybook-static -l 6006" http://localhost:6006

- name: Run VRT comparison
  run: bun run vrt
  continue-on-error: true

- name: Upload VRT diff
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: vrt-diff
    path: frontend/.reg/
    retention-days: 7
```

VRT のステップに `continue-on-error: true` を付けているのは、視覚的な差分は必ずしもビルド失敗ではなく意図的な変更の場合があるため。差分があれば `.reg/` ディレクトリをアーティファクトに上げて確認できるようにしている。

### test-coverage.yml — カバレッジ計測

MySQL サービスを立ち上げ、フロントエンドとバックエンドのカバレッジを同一ジョブで計測する。

```yaml
services:
  mysql:
    image: mysql:8.0
    env:
      MYSQL_DATABASE: diary_db
    ports:
      - 3306:3306
    options: >-
      --health-cmd="mysqladmin ping -h 127.0.0.1 --silent"
      --health-interval=10s
      --health-retries=5
```

それぞれの coverage ステップに `continue-on-error: true` を付け、最後にどちらかが失敗していたら `exit 1` する構成にしている。こうすることでフロントエンド・バックエンド両方のカバレッジレポートを必ずアップロードしてから失敗を通知できる。

```yaml
- name: Fail when coverage failed
  if: steps.frontend_coverage.outcome == 'failure' || steps.backend_coverage.outcome == 'failure'
  run: exit 1
```

---

## 不要なワークフローファイルを削除

整備前には古い時代のワークフローファイルが残っており、新旧の定義が混在していた。以下の 3 ファイルを削除した。

- `backend.yaml`
- `backend-ci.integration.yaml`
- `frontend.yaml`

これらは現在のモノレポ構成・Bun 移行に対応していないため、残しておくと混乱の元になる。

---

## ワークフロー設計のまとめ

| ワークフロー | トリガー | 実行内容 | 目的 |
|---|---|---|---|
| `ci.yml` | push to main/develop | lint + typecheck + small + build | 高速フィードバック |
| `ci-pr.yml` | pull_request | lint + small + medium + E2E + Storybook | PR の品質ゲート |
| `ci-nightly.yml` | cron (JST 03:00) | 全テスト + VRT | 深夜の全量検証 |
| `test-coverage.yml` | push/PR to main/develop | カバレッジ計測（MySQL 付き）| 品質指標の可視化 |
| `copilot-setup-steps.yml` | 変更時 / 手動 | bun install のみ | Copilot エージェント環境 |

---

## 移行のポイントまとめ

1. **`actions/checkout@v4` を使う** — v5 は存在しない
2. **`oven-sh/setup-bun@v1` + `bun install --frozen-lockfile`** — npm/Node.js をそのまま使わない
3. **スクリプト名は `package.json` で確認する** — CI と実装の乖離は起動時にしか気づけない
4. **DB 環境変数は `.env.example` と一致させる** — 変数名の不一致はランタイムエラーとして顕在化する
5. **`if: always()` でアーティファクトを取りこぼさない** — テスト失敗後でもレポートを見られるようにする
6. **VRT は `continue-on-error: true` にする** — 意図的な変更と意図しない変更を人間が判断できる余地を持たせる

## まとめ

これらは現在のモノレポ構成・Bun 移行に対応していないため、残しておくと混乱の元になる。
