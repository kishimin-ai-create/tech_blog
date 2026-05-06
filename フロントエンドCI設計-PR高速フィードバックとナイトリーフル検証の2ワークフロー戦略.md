# フロントエンド CI 設計 — PR 高速フィードバックとナイトリーフル検証の 2 ワークフロー戦略

## 対象読者

- GitHub Actions でフロントエンドの CI を整備しようとしている
- Playwright のテストが増えてきて、PR のたびに全ブラウザ・全テストを実行するのが遅いと感じている
- Storybook ビルドや E2E テストをどのタイミングで走らせるか悩んでいる

## この記事が扱う範囲

- `ci-pr.yml`（PR トリガー）と `ci-nightly.yml`（夜間スケジュール）への 2 分割設計
- 各ワークフローのステップ構成とタイムアウト設計
- `playwright.config.ts` での `PLAYWRIGHT_BASE_URL` 環境変数対応
- 旧 `playwright.yml` との比較

reg-suit によるビジュアルリグレッション統合の詳細は別記事で扱う。

---

## 背景：シングルワークフローの限界

以前のリポジトリには `playwright.yml` という 1 本のワークフローが存在していた。

```yaml
# 旧: .github/workflows/playwright.yml（削除済み）
on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]

jobs:
  test:
    timeout-minutes: 60
    steps:
      - name: Install Playwright Browsers
        run: npx playwright install --with-deps   # 全ブラウザ
      - name: Run Playwright tests
        run: npx playwright test                  # 全テスト・全ブラウザ
```

このワークフローには以下の問題があった。

- **lint / typecheck / Vitest が含まれていない**：型エラーや ESLint 違反がある状態でも E2E が走る
- **全ブラウザ × 全テストで常に 60 分タイムアウト**：PR のたびに重い処理が走る
- **Storybook ビルド確認がない**：Storybook の壊れを CI で検知できない
- **reg-suit ビジュアルリグレッションを組み込む余地がない**：毎 PR で実行するには重すぎる

これらを解消するために、CI を「PR 用（高速）」と「ナイトリー（フル）」に分割した。

---

## 2 ワークフローの役割分担

| 観点 | ci-pr.yml（PR 用） | ci-nightly.yml（夜間） |
|---|---|---|
| トリガー | `push` (main/develop) + `pull_request` | 毎日 JST 03:00 + `workflow_dispatch` |
| タイムアウト | 15 分 | 60 分 |
| Lint・型チェック | ✅ | ✅（暗黙） |
| Vitest | ✅ | — |
| Storybook ビルド | ✅ | ✅ |
| Playwright | Chromium のみ・`@smoke` タグのみ | 全ブラウザ・全テスト |
| ビジュアルリグレッション | — | ✅（reg-suit） |
| アーティファクト保持 | 7 日 | 30 日 |

PR は **15 分以内で完了する高速フィードバック**を最優先し、フル検証は夜間に委ねる設計だ。

---

## ci-pr.yml の詳細

```yaml
# .github/workflows/ci-pr.yml
name: CI (PR)

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    defaults:
      run:
        working-directory: frontend

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run typecheck

      - name: Run unit & integration tests
        run: npm run test

      # Storybook が壊れていないか確認
      - name: Build Storybook
        run: npm run build-storybook -- --quiet

      # Chromium のみ・@smoke タグのみ（テストがなくてもパス）
      - name: Install Playwright Browsers
        run: npx playwright install --with-deps chromium

      - name: Run Playwright smoke tests
        run: npx playwright test --grep "@smoke" --pass-with-no-tests --project=chromium

      - name: Upload Playwright report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: frontend/playwright-report/
          retention-days: 7
```

### 設計のポイント

**`--grep "@smoke" --pass-with-no-tests`**  
`@smoke` タグが付いたテストのみ実行する。`--pass-with-no-tests` を付けているため、まだ `@smoke` テストが 1 本も存在しない段階でもワークフローが失敗しない。テストを育てながら段階的に CI を強化できる。

**Chromium のみ**  
Firefox や WebKit のインストールを省くことでジョブ時間を大きく短縮できる。クロスブラウザ確認は夜間ジョブに任せる。

**Storybook ビルドを PR 時に含める理由**  
Storybook は実装コードに直接依存するため、コンポーネントの型変更や import 削除の影響をすぐに受ける。ビルド確認だけなら数分で終わるため PR 時に含めても 15 分の制約に収まる。

---

## ci-nightly.yml の詳細

```yaml
# .github/workflows/ci-nightly.yml
name: CI (Nightly)

on:
  schedule:
    - cron: '0 18 * * *'  # UTC 18:00 = JST 03:00
  workflow_dispatch:

jobs:
  e2e-full:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    defaults:
      run:
        working-directory: frontend

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # reg-keygen-git-hash-plugin に必要

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: npm ci

      # 本番ビルド → vite preview でサーバー起動（ポート 4173）
      - name: Build app
        run: npm run build

      - name: Start preview server
        run: npm run preview &

      - name: Wait for preview server
        run: npx --yes wait-on http://localhost:4173 --timeout 30000

      # 全ブラウザ・全テスト
      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Run Playwright (full suite, all browsers)
        run: npx playwright test
        env:
          PLAYWRIGHT_BASE_URL: http://localhost:4173

      - name: Build Storybook
        run: npm run build-storybook -- --quiet

      # ビジュアルリグレッション（詳細は別記事）
      - name: Take visual regression screenshots
        run: npx playwright test --grep "@visual" --project=chromium
        env:
          PLAYWRIGHT_BASE_URL: http://localhost:4173

      - name: Run reg-suit
        run: npx reg-suit run
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Upload Playwright report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: frontend/playwright-report/
          retention-days: 30
```

### 設計のポイント

**`vite build` → `npm run preview`**  
開発サーバー（`npm run dev`）ではなく本番ビルドのプレビューサーバーを使う。`dev` サーバーは本番ビルドとバンドル挙動が異なるため、夜間ジョブでは本番に近い環境で E2E を実行することを選択した。

**`wait-on` でサーバー起動を待つ**  
バックグラウンドで起動したサーバーが実際に応答できる状態になるまで Playwright を走らせないよう、`wait-on` で HTTP チェックをしている。30 秒のタイムアウトを設けている。

**`fetch-depth: 0`**  
reg-keygen-git-hash-plugin は git の全コミット履歴からハッシュを生成するため、デフォルトの shallow clone（`fetch-depth: 1`）では動作しない。`fetch-depth: 0` で full clone に切り替えている。

**`workflow_dispatch`**  
スケジュール実行に加えて手動トリガーも用意している。リリース前の確認や reg-suit のベースライン取得など、任意のタイミングで手動実行できる。

---

## playwright.config.ts への変更

```typescript
use: {
  /* Base URL to use in actions like `await page.goto('')`.
   * Overridden by PLAYWRIGHT_BASE_URL in CI (set by ci-nightly.yml). */
  baseURL: process.env['PLAYWRIGHT_BASE_URL'] ?? 'http://localhost:4173',
```

以前は `baseURL` がコメントアウトされていた。`PLAYWRIGHT_BASE_URL` 環境変数があればそれを使い、なければ `localhost:4173`（vite preview のデフォルトポート）にフォールバックする形にした。

これにより：
- ローカル実行：環境変数なし → `localhost:4173`（`npm run preview` を事前に起動しておく）
- CI（nightly）：`PLAYWRIGHT_BASE_URL=http://localhost:4173` を明示的に渡す

CI 側でサーバーを立ち上げる方式を選択しているため、`playwright.config.ts` の `webServer` セクションはコメントアウトのままにしている。

---

## 設計上のトレードオフ

### PR で全ブラウザ E2E を走らせない理由

Firefox・WebKit のインストールだけで数分かかる。PR のたびにこれを繰り返すと「コミットしてから結果を見るまでの時間」が伸び、開発者の集中が途切れる。クロスブラウザの差異は日次で検出できれば運用上は十分と判断した。

### PR に `--pass-with-no-tests` を入れる理由

CI ファイルを先に整備し、テスト実装はあとから追加していく開発スタイルと相性がよい。インフラが整ってからテストを書くというサイクルで、CI が「テストがないから失敗」というノイズを出し続けないようにしている。

### 夜間ジョブを JST 03:00 にした理由

UTC 18:00 は日本時間で深夜 3 時。開発者が始業する前に結果が出ており、朝一番のスタンドアップで前日の夜間テスト結果を確認できる。

---

## まとめ

- **旧 `playwright.yml`** を廃止し、**`ci-pr.yml`（15 分・高速）** と **`ci-nightly.yml`（60 分・フル）** に分割した
- PR では lint・型チェック・Vitest・Storybook ビルド・Playwright smoke（Chromium のみ）を実行し、開発者フィードバックを高速に保つ
- 夜間では本番ビルドのプレビューサーバーを立ち上げ、全ブラウザ E2E + Storybook ビルド + ビジュアルリグレッションをフル実行する
- `PLAYWRIGHT_BASE_URL` 環境変数で CI とローカルの接続先を統一的に制御できる
- reg-suit ビジュアルリグレッションの統合詳細は別記事を参照
