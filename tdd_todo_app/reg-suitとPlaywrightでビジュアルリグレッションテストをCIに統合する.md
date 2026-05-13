# reg-suit と Playwright でビジュアルリグレッションテストを CI に統合する

## 対象読者

- フロントエンドの UI 変更を目視確認に頼っており、意図しないレイアウト崩れを自動検出したい
- reg-suit を使ったビジュアルリグレッションを GitHub Actions に組み込もうとしている
- storycap を試したが Storybook 10 で動かなかった経験がある、または回避策を探している

## この記事が扱う範囲

- storycap が Storybook 10 で動作しない問題とその回避策
- Playwright でスクリーンショットを取得し reg-suit に渡す構成
- `regconfig.json` の設計（閾値・S3 連携・GitHub PR コメント）
- `ci-nightly.yml` への統合方法とシークレット管理

CI ワークフロー全体の設計（PR 用と夜間の分割戦略）は別記事で扱う。

---

## ビジュアルリグレッションテストの目的

ビジュアルリグレッションテストとは、UI のスクリーンショットを前回と比較して「意図しない見た目の変化」を検出する手法だ。テキスト差分（Jest や Vitest）では検知できないレイアウトの崩れ・色変化・フォントサイズの変更などを機械的に捕まえられる。

---

## storycap と Storybook 10 の非互換問題

reg-suit の公式ドキュメントや多くのブログ記事では、Storybook の各ストーリーを自動的にスクリーンショットする **storycap** との組み合わせが紹介されている。

しかし、このプロジェクトは **Storybook 10** を使用しており、storycap はこのバージョンに対応していない。storycap の npm ページや GitHub Issue を確認すると、Storybook 8 以降の互換性が保証されておらず、Storybook 10 では起動自体が失敗する。

```
# storycap をインストールして実行すると...
Error: Cannot find module '@storybook/core-server'
# あるいは起動後にスクリーンショットが取得されない
```

Storybook 10 は 2025 年にリリースされた比較的新しいメジャーバージョンであり、周辺ツールの対応が追いついていない。

### 回避策：Playwright でスクリーンショットを取得する

storycap を使わずに reg-suit を使う方法として、**Playwright でスクリーンショットを取得して `.reg/actual/` に保存する**アプローチを選択した。

reg-suit は「比較対象のスクリーンショットが `.reg/actual/` に存在すること」だけを前提としており、スクリーンショットの取得方法を問わない。つまり Playwright で撮って配置すれば、storycap と同等の役割を果たせる。

---

## 構成ファイルの設計

### e2e/visual.spec.ts

```typescript
import { test } from '@playwright/test';
import fs from 'node:fs';
import path from 'node:path';

// スクリーンショットを .reg/actual/ に保存する（reg-suit の比較対象）
const ACTUAL_DIR = path.join(process.cwd(), '.reg', 'actual');

test.beforeAll(() => {
  fs.mkdirSync(ACTUAL_DIR, { recursive: true });
});

test.describe('@visual', () => {
  test('home page', async ({ page }) => {
    await page.goto('/');
    await page.waitForLoadState('networkidle');
    await page.screenshot({
      path: path.join(ACTUAL_DIR, 'home.png'),
      fullPage: true,
    });
  });
});
```

**`@visual` タグ（`test.describe('@visual', ...)`）**  
CI で `--grep "@visual"` を指定することで、ビジュアルリグレッション用のスクリーンショット取得ステップだけを分離して実行できる。通常の E2E テストと混在させずに、必要なときだけ動かす設計だ。

**`waitForLoadState('networkidle')`**  
ページの初期レンダリングが完了し、ネットワーク通信が落ち着いてからスクリーンショットを取る。非同期でデータを取得するコンポーネントが存在する場合、ローディングスピナーが写り込むことを防ぐ。

**`fullPage: true`**  
ビューポートに収まらないコンテンツも含めて全体をキャプチャする。スクロールが必要な部分のレイアウト崩れも検出できる。

**`.reg/actual/` への直接保存**  
Playwright 標準のスナップショット（`expect(page).toHaveScreenshot()`）ではなく、`page.screenshot()` でパスを明示的に指定している。これにより reg-suit が期待するディレクトリ構造に直接配置できる。

---

### regconfig.json

```json
{
  "core": {
    "workingDir": ".reg",
    "actualDir": ".reg/actual",
    "thresholdRate": 0.01,
    "ximgdiff": {
      "invocationType": "client"
    }
  },
  "plugins": {
    "reg-keygen-git-hash-plugin": true,
    "reg-publish-s3-plugin": {
      "bucketName": "REPLACE_WITH_YOUR_BUCKET_NAME"
    },
    "reg-notify-github-plugin": {
      "prComment": true
    }
  }
}
```

**`thresholdRate: 0.01`（1%）**  
ピクセル単位の差分がスクリーンショット全体の 1% 以内であれば合格とする。アンチエイリアスや微妙なサブピクセルレンダリングの違いで誤検知が発生しないよう、わずかな誤差を許容している。0 にすると 1 ピクセルの差異でも失敗となり、ノイズが増える。

**`reg-keygen-git-hash-plugin`**  
git のコミットハッシュをキーとして使うプラグイン。「このコミットのスクリーンショット」と「前のコミットのスクリーンショット」を紐付けて比較する。このプラグインが機能するには git の全履歴が必要なため、後述するように CI で `fetch-depth: 0` が必須になる。

**`reg-publish-s3-plugin`**  
スクリーンショットと差分レポートを Amazon S3 に保存する。これにより異なる CI ジョブ間（ベースライン取得ジョブと比較ジョブ）でデータを共有できる。`bucketName` は CI 実行時に GitHub Secrets から動的に注入するため、リポジトリにはプレースホルダーを置いている。

**`reg-notify-github-plugin`（`prComment: true`）**  
差分が検出されたとき PR に自動でコメントを投稿する。レビュワーがレポートのリンクを受け取り、視覚的な差分を確認できる。

---

## .gitignore への追加

```gitignore
# reg-suit
/.reg/
```

`.reg/` ディレクトリには実際のスクリーンショット（実行ごとに変わる）と差分レポートが保存される。これらはバイナリファイルかつ大容量になりやすく、S3 で管理するため git には含めない。

---

## ci-nightly.yml への統合

reg-suit を夜間ジョブに組み込んだ部分を抜粋する。

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # reg-keygen-git-hash-plugin には全履歴が必要

# --- 前段：ビルドとサーバー起動（省略） ---

# ステップ 1：@visual タグのテストだけ実行してスクリーンショット取得
- name: Take visual regression screenshots
  run: npx playwright test --grep "@visual" --project=chromium
  env:
    PLAYWRIGHT_BASE_URL: http://localhost:4173

# ステップ 2：bucketName を GitHub Secret から動的に注入
- name: Inject S3 bucket name into regconfig
  run: |
    jq --arg bucket "$REG_SUIT_BUCKET_NAME" \
      '.plugins["reg-publish-s3-plugin"].bucketName = $bucket' \
      regconfig.json > tmp.json && mv tmp.json regconfig.json
  env:
    REG_SUIT_BUCKET_NAME: ${{ secrets.REG_SUIT_BUCKET_NAME }}

# ステップ 3：reg-suit 実行（比較・S3 アップロード・PR コメント投稿）
- name: Run reg-suit
  run: npx reg-suit run
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

# ステップ 4：レポートをアーティファクトとして保存
- name: Upload reg-suit report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: reg-suit-report
    path: frontend/.reg/
    retention-days: 30
```

### `jq` で bucketName を動的に注入する理由

`regconfig.json` にバケット名をハードコードするとリポジトリにバケット名が露出する。GitHub Secrets に `REG_SUIT_BUCKET_NAME` を設定し、`jq` で実行時に書き換えることで秘匿情報をコードに含めずに済む。

`regconfig.json` の `bucketName` フィールドは `REPLACE_WITH_YOUR_BUCKET_NAME` というプレースホルダーにしてあり、このステップが実行されるまで実際の値は入らない。

### 必要な GitHub Secrets

| シークレット名 | 用途 |
|---|---|
| `AWS_ACCESS_KEY_ID` | S3 バケットへのアップロード権限 |
| `AWS_SECRET_ACCESS_KEY` | 同上 |
| `REG_SUIT_BUCKET_NAME` | スクリーンショット保存先の S3 バケット名 |
| `GITHUB_TOKEN` | PR へのコメント投稿（GitHub Actions 標準で自動設定される） |

`GITHUB_TOKEN` は Actions が自動で発行するためリポジトリ設定は不要。`AWS_*` と `REG_SUIT_BUCKET_NAME` はリポジトリの Settings → Secrets に追加する必要がある。

---

## インストールしたパッケージ

```json
{
  "devDependencies": {
    "reg-suit": "^0.14.5",
    "reg-keygen-git-hash-plugin": "^0.14.5",
    "reg-publish-s3-plugin": "^0.14.4",
    "reg-notify-github-plugin": "^0.14.5"
  }
}
```

`storycap` はインストールしていない。Storybook 10 との互換性がなく、インストールしても動作しないためだ。

---

## 動作フローのまとめ

```
[夜間 CI ジョブ]
  ↓
vite build → npm run preview（ポート 4173）
  ↓
Playwright @visual テスト実行（Chromium のみ）
  → .reg/actual/home.png 等が生成される
  ↓
jq で regconfig.json に bucketName を注入
  ↓
npx reg-suit run
  → 前回実行の画像を S3 から取得
  → 差分を計算（閾値 1% 超なら差分あり）
  → 今回の画像と差分レポートを S3 にアップロード
  → 差分ありの場合 PR にコメントを投稿
  ↓
.reg/ をアーティファクトとして保存（30 日間）
```

---

## 注意点

### Chromium 限定にしている理由

スクリーンショットのピクセルレベル比較はブラウザエンジンによってレンダリング結果が微妙に異なる。Firefox と WebKit を混在させると「実際には同じデザインなのに差分として検出される」誤検知が増える。ビジュアルリグレッションは 1 ブラウザに統一することで安定性が保たれる。

### ベースラインの初回登録

reg-suit を初めて実行した時点では S3 にベースライン画像が存在しないため、「比較対象なし」として処理される。`workflow_dispatch` で手動実行して初回のベースラインを登録したあと、以降の実行から差分比較が機能するようになる。

### ローカルでの動作確認

`.reg/` は `.gitignore` に追加しているため、ローカルで `npx playwright test --grep "@visual"` を実行してもスクリーンショットはリポジトリに含まれない。ローカル確認は `actual/` の画像ファイルを直接目視するか、`npx reg-suit run` をローカル実行して確認する。

---

## まとめ

- **storycap は Storybook 10 に非対応**。reg-suit 連携に storycap を使えない場合は Playwright で代替できる
- Playwright の `@visual` タグ付きテストでスクリーンショットを `.reg/actual/` に出力し、`npx reg-suit run` で比較・通知する
- `thresholdRate: 0.01` で微細なレンダリング差異による誤検知を防ぐ
- S3 バケット名は `jq` で実行時注入することで秘匿情報をコードから排除する
- `fetch-depth: 0` を忘れると reg-keygen-git-hash-plugin がハッシュを解決できない
- ビジュアルリグレッションは夜間ジョブに組み込み、PR 時の高速フィードバックを妨げない設計とする
