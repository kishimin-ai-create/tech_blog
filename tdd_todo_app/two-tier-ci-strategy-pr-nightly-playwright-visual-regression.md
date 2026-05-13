# PRで高速フィードバック、夜間でフル検証 — フロントエンドCI 2ワークフロー戦略とPlaywright視覚回帰テスト

## 対象読者

- GitHub Actions でフロントエンド CI を整備しており、PR のたびに全ブラウザ・全テストを実行するのが遅くなってきた人
- Playwright の E2E テストを CI に組み込みたいが、実行時間と信頼性のバランスに悩んでいる人
- `toHaveScreenshot()` による視覚回帰テストを導入したい人

---

## 背景

以前のリポジトリには `playwright.yml` という単一のワークフローが存在していた。PR のたびに全ブラウザ・全テストを実行していたため、次の問題があった。

- **実行時間が長い** — Chromium/Firefox/WebKit の3ブラウザで全テストを走らせると、PR のフィードバックに時間がかかる
- **視覚回帰テストと機能テストが混在** — スクリーンショット比較はフォントレンダリングなど環境依存の要因でフレーキーになりやすく、PR ブロッカーには向かない
- **一回の実行で全責任を負う** — 何か落ちたときにどのカテゴリの問題かわかりにくい

この問題を解消するため、CI を **PR 向けの高速フィードバックワークフロー** と **夜間のフル検証ワークフロー** に分割した。

---

## 2ワークフローの設計

| | `ci-pr.yml` | `ci-nightly.yml` |
|---|---|---|
| **トリガー** | push (main/develop) + PR | schedule `0 18 * * *`（JST 03:00）+ workflow_dispatch |
| **タイムアウト** | 15分 | 60分 |
| **ブラウザ** | Chromium のみ | Chromium + Firefox + WebKit |
| **Playwright 範囲** | `@smoke` タグのみ | 全テスト（視覚テスト除く） + `@visual` 専用ステップ |
| **Storybook** | ビルド確認あり | ビルド確認あり |
| **アーティファクト保存期間** | 7日 | 30日 |

---

## ci-pr.yml — PR の高速フィードバック

```yaml
# .github/workflows/ci-pr.yml（ステップ部分）
- name: Lint
  run: npm run lint

- name: Type check
  run: npm run typecheck

- name: Run unit & integration tests
  run: npm run test

- name: Build Storybook
  run: npm run build-storybook -- --quiet

- name: Install Playwright Browsers
  run: npx playwright install --with-deps chromium

- name: Run Playwright smoke tests
  run: npx playwright test --grep "@smoke" --project=chromium
```

PR ワークフローで特に注目したいのが `@smoke` タグによる絞り込みだ。全 E2E テストではなく、最も重要な導線だけを Chromium 一本で実行することで 15分以内の完了を目指している。

### `--pass-with-no-tests` を削除した理由

当初の実装には `--pass-with-no-tests` フラグがあった。

```yaml
# 修正前（問題あり）
run: npx playwright test --grep "@smoke" --pass-with-no-tests --project=chromium
```

このフラグは、`@smoke` タグを持つテストがひとつもないときでも CI を成功扱いにする。これは**無言でのカバレッジ消失**を引き起こす。例えばリファクタリングで全テストからタグを消してしまっても、CI は緑のままになる。

修正では `--pass-with-no-tests` を削除し、`frontend/e2e/example.spec.ts` の既存テストに `@smoke` タグを追加することで「少なくとも1つのスモークテストが常に存在する」状態を保証した。

```ts
// frontend/e2e/example.spec.ts
test('has title @smoke', async ({ page }) => {
  await page.goto('https://playwright.dev/');
  await expect(page).toHaveTitle(/Playwright/);
});
```

---

## ci-nightly.yml — 夜間のフル検証

夜間ワークフローは **本番ビルドに対して全ブラウザでテストを実行** するため、`vite preview` で静的サーバーを起動してから Playwright を走らせる。

```yaml
- name: Build app
  run: npm run build

- name: Start preview server
  run: npm run preview &

- name: Wait for preview server
  run: npx --yes wait-on http://localhost:4173 --timeout 30000

- name: Run Playwright (full suite, all browsers)
  run: npx playwright test --grep-invert "@visual"
  env:
    PLAYWRIGHT_BASE_URL: http://localhost:4173

- name: Run visual regression tests
  run: npx playwright test --grep "@visual" --project=chromium
  env:
    PLAYWRIGHT_BASE_URL: http://localhost:4173
```

### `@visual` テストを分離した理由

コードレビューで「フル Playwright 実行に `@visual` テストが含まれており、その後さらに `@visual` 専用ステップが実行されているため2重実行になっている」という指摘があった。

修正として、フル実行ステップに `--grep-invert "@visual"` を追加した。これにより:

- フル実行ステップ: `@visual` **以外** の全テストを全ブラウザで実行
- 視覚テストステップ: `@visual` テストのみを Chromium で実行

視覚テストを Chromium 専用にするのは、フォントレンダリングの差異によるブラウザ間のスナップショット不一致を避けるためだ。

### `workflow_dispatch` による手動実行

夜間ワークフローには `workflow_dispatch` トリガーも設定されている。これにより:

- スナップショットのベースライン更新が必要なとき
- フレーキーなテストのデバッグをしたいとき

03:00 JST のスケジュールを待たずに手動で実行できる。

---

## Playwright 視覚回帰テストの設定

### `playwright.config.ts` の設定

```ts
export default defineConfig({
  // ...
  expect: {
    toHaveScreenshot: {
      maxDiffPixelRatio: 0.01, // 1%以内のピクセル差は許容
      animations: 'disabled',  // アニメーションを止めてスナップショットを安定化
    },
  },
  use: {
    baseURL: process.env['PLAYWRIGHT_BASE_URL'] ?? 'http://localhost:4173',
    trace: 'on-first-retry',
  },
});
```

`maxDiffPixelRatio: 0.01` は「全ピクセルの1%以内の差異は許容する」という設定だ。アンチエイリアスの微細な差や CSS レンダリングの揺れを吸収しつつ、意図しないレイアウト崩れは検出できる。

`animations: 'disabled'` はスクリーンショット比較においてアニメーション中間状態のキャプチャを防ぐために必須だ。これがないとアニメーションのタイミング次第で差分が出てフレーキーになる。

### 視覚テストのコード

```ts
// frontend/e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test.describe('@visual', () => {
  test('home page', async ({ page }) => {
    await page.goto('/');
    await page.waitForLoadState('networkidle');
    await expect(page).toHaveScreenshot('home.png', { fullPage: true });
  });
});
```

`waitForLoadState('networkidle')` でネットワーク通信が落ち着いてからスクリーンショットを撮ることで、非同期データ取得中の不安定な状態をキャプチャしない。

スナップショットは `e2e/__snapshots__/` にコミットされている。これにより CI がローカルのスナップショットを持ち込まずに比較できる。

### スナップショットの更新手順

意図的な UI 変更があった場合は、変更と同じ PR でスナップショットを更新する。

```bash
npx playwright test --update-snapshots
```

---

## CI 失敗時のデバッグ

視覚テストが失敗したとき、どのピクセルが変わったかを確認するために夜間ワークフローは差分アーティファクトをアップロードする。

```yaml
- name: Upload snapshot diffs
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: snapshot-diffs
    path: frontend/test-results/
    retention-days: 7
```

GitHub Actions のアーティファクトからダウンロードすると、before/after/diff の3枚が確認できる。

---

## まとめ

| 設計ポイント | 効果 |
|---|---|
| PR ワークフローを 15分以内に収める | 開発中のフィードバックループを短縮 |
| `@smoke` タグで重要な導線のみを PR で実行 | 全 E2E を走らせずに主要な壊れを検出 |
| `--pass-with-no-tests` の削除 | 無言でのカバレッジ消失を防止 |
| `@visual` テストを夜間専用に分離 | フレーキーなスナップショット比較が PR をブロックしない |
| `--grep-invert "@visual"` でフル実行から除外 | 2重実行による時間とアーティファクトの無駄遣いを排除 |
| 失敗時に差分アーティファクトをアップロード | 原因調査のコストを下げる |

CI のコスト（時間・安定性）と検出力（カバレッジ・品質）のバランスを、テストカテゴリの分離という手法で調整した設計だ。
