# コードレビュー対応: orval / MSW / CI フロントエンドセットアップの修正まとめ

## 対象読者

- orval + MSW v2 + Vitest の構成を導入しようとしているフロントエンドエンジニア
- GitHub Actions で Playwright の E2E テストを CI に組み込みたい方
- コードレビューで指摘された問題の「なぜ直すのか」まで理解したい方

## 本記事のスコープ

- **対象**: `frontend/src/api/client.ts`、`frontend/src/test/setup.ts`、`frontend/playwright.config.ts`、`.github/workflows/ci-pr.yml`、`.github/workflows/ci-nightly.yml` に加えた修正
- **対象外**: ビジュアルリグレッションのベースラインスナップショット生成（プラットフォーム依存のため別途対応が必要）、生成ファイルの `.gitignore` 移行（大規模変更のため後続 PR）

---

## 背景

orval によるコード生成・MSW v2 を使った Vitest 統合・Playwright による E2E テストを含むフロントエンドブランチに対して、2 回のコードレビューが実施された。それぞれのレビューで発見された計 6 件の知見を処理した。P1 が 4 件、P2/P3 が 2 件で、最終的に 5 件をコード修正で解消し、残り 1 件はアーキテクチャ上の判断として reply-only とした。

---

## 修正 1: `customFetch` が orval 生成型の HTTP エンベロープを返すよう変更

### 問題 1: 非 JSON エラーレスポンスで SyntaxError が発生する

最初のレビューで指摘されたのは、`customFetch` がエラーレスポンスの本文を無条件に `response.json()` でパースしていた点だ。

```ts
// 修正前
if (!response.ok) {
  throw (await response.json()) as unknown; // ← 非 JSON 本文で SyntaxError
}
```

502 Bad Gateway や 504 Gateway Timeout のような HTML レスポンスが来ると `response.json()` が `SyntaxError` をスローし、HTTP ステータスコードも URL も失われたまま呼び出し元に届く。

### 問題 2: orval 生成型が期待するエンベロープと型が合わない

2 回目のレビューでは、より根本的な問題が指摘された。orval が生成する型は `{ data, status, headers }` 形式のエンベロープを期待しているが、`customFetch` が `response.json()` をそのまま返していたため **ランタイムの値と生成された型が不一致**になっていた。

```ts
// orval 生成型の例（抜粋）
export type PostApiV1AppsResponse = {
  data: AppResponse;
  status: number;
  headers: Headers;
};
```

この場合、生成されたフックから `result.status` を参照しても `undefined` が返るため、レスポンスによる分岐ロジックが壊れる。

### 最終的な修正

`response.ok` ガードとスロー処理を撤廃し、**すべての HTTP レスポンスに対してエンベロープを返す**設計に変更した。4xx / 5xx も orval 生成型では typed response variant として扱われるため、例外ではなく値として返すのが正しい。

```ts
// frontend/src/api/client.ts（修正後）
export const customFetch = async <T>(
  url: string,
  options?: RequestInit,
): Promise<T> => {
  const response = await fetch(`${BASE_URL}${url}`, options);

  let data: unknown;
  try {
    data = await response.json();
  } catch {
    data = await response.text(); // 非 JSON 本文のフォールバック
  }

  return { data, status: response.status, headers: response.headers } as T;
};
```

**ポイント**:
- JSON パースを `try/catch` で囲み、失敗時は `response.text()` にフォールバック
- 成功・失敗問わず `{ data, status, headers }` を返すことで生成型との契約を満たす
- 例外を投げなくなったため、呼び出し側は `status` フィールドで成否を判定する

---

## 修正 2: MSW の `onUnhandledRequest` を `'warn'` から `'error'` に変更

### 問題

```ts
// 修正前
server.listen({ onUnhandledRequest: 'warn' });
```

`'warn'` モードでは、ハンドラーの定義がない API 呼び出しがあっても警告ログが出るだけでテストは続行する。jsdom 内部でネットワークエラーが発生することもあるが、コンポーネントがそれをうまくハンドリングしていれば **テストが誤ってパスする**ことがある。CI ログの警告は見落とされやすく、カバレッジの漏れが無音で蓄積する。

### 修正

```ts
// frontend/src/test/setup.ts（修正後）
server.listen({ onUnhandledRequest: 'error' });
```

ハンドラーのない API 呼び出しが発生した瞬間にテストが即座に失敗する。これにより、新しいフックを追加してもそれに対応する MSW ハンドラーを書き忘れた場合に、すぐ CI で検知できる。

---

## 修正 3: `@testing-library/jest-dom` を devDependencies に明示追加

### 問題

`frontend/src/test/setup.ts` では `import '@testing-library/jest-dom/vitest'` を使っているが、`package.json` の `devDependencies` に明示されていなかった。今は推移的依存としてインストールされているが、親パッケージがその依存を削除したタイミングで突然壊れる。

### 修正

```bash
npm install -D @testing-library/jest-dom
```

`@testing-library/jest-dom@^6.9.1` を `devDependencies` に追加し、`npm ci` による確定的なインストールを保証した。

---

## 修正 4: `playwright.config.ts` の `webServer` ブロックをアンコメント

### 問題

```ts
// 修正前（コメントアウトされた状態）
// webServer: {
//   command: 'npm run preview',
//   url: 'http://localhost:4173',
//   reuseExistingServer: !process.env.CI,
// },
```

`webServer` がコメントアウトされているため、ローカルで `npx playwright test` を実行するには事前に `npm run build && npm run preview` を手動で起動する必要があった。サーバーが立っていない状態だとすべてのテストが `connection refused` で落ちるが、エラーメッセージが直感的でないため原因に気づきにくい。

### 修正

```ts
// frontend/playwright.config.ts（修正後）
webServer: {
  command: 'npm run preview',
  url: 'http://localhost:4173',
  reuseExistingServer: !process.env.CI,
},
```

`reuseExistingServer: !process.env.CI` の設計が重要:
- **ローカル**: `CI` 環境変数が未設定なので `reuseExistingServer: true` → すでに起動中のサーバーを再利用
- **CI**: `CI=true` なので `reuseExistingServer: false` → Playwright が必ず新しいサーバーを起動

CI ワークフロー側への変更は不要で、後述のビルドステップ追加と組み合わせることで動作する。

---

## 修正 5: CI (PR) に `npm run build` ステップを追加

### 問題

`ci-pr.yml` では Playwright 実行前に `npm run build` が存在しなかった。

Playwright の `webServer` が `npm run preview` を起動するが、`vite preview` は `dist/` ディレクトリを配信するコマンドであり、**`npm run build` でビルドされていないと配信対象がない**。クリーンな CI ランナーで `dist/` が存在しないため、すべての smoke テストが実行前に失敗する。

### 修正

```yaml
# .github/workflows/ci-pr.yml（追加ステップ）
# Vite build (dist/ must exist before vite preview can serve)
- name: Build frontend
  run: npm run build

# Playwright（@smoke タグのみ・Chromium のみ）
- name: Install Playwright Browsers
  run: npx playwright install --with-deps chromium

- name: Run Playwright smoke tests
  run: npx playwright test --grep "@smoke" --project=chromium
```

`Build frontend` を Playwright 実行直前に挿入することで、`webServer` が `dist/` を確実に配信できる状態になる。

---

## 修正 6: CI (Nightly) の重複 `preview` サーバー起動を削除

### 問題

以前の `ci-nightly.yml` には次のようなステップが存在していた（修正前の姿）:

```yaml
- name: Start preview server
  run: npm run preview &

- name: Wait for preview server
  run: npx wait-on http://localhost:4173
```

一方、`playwright.config.ts` の `webServer` も `npm run preview` を起動し、CI では `reuseExistingServer: false` になっている。Playwright の仕様として、`reuseExistingServer: false` のときにポートがすでに使用中だと **起動エラーをスローする**。そのため、手動で先に起動したサーバーが存在していると Playwright がすべてのテストを実行する前に落ちる。

### 修正

手動の `Start preview server` と `Wait for preview server` ステップを削除し、**Playwright の `webServer` 設定がサーバーライフサイクルを一元管理する**設計にした。

```yaml
# ci-nightly.yml（修正後の関連部分）
- name: Build app
  run: npm run build   # dist/ を生成してから webServer が起動できる

- name: Install Playwright Browsers
  run: npx playwright install --with-deps

- name: Run Playwright (full suite, all browsers)
  run: npx playwright test --grep-invert "@visual"
```

`Build app` ステップは残したまま `webServer` 管理に委ねることで、二重起動の競合を解消している。

---

## Reply-only: 対応しなかった指摘とその理由

### ビジュアルリグレッションのベースライン未コミット

`ci-nightly.yml` で `npx playwright test --grep "@visual"` を実行しているが、比較対象のベースライン画像が存在しないため初回実行時は必ず失敗する。

**対応しなかった理由**: ベースラインは OS のフォントレンダリングに依存するため、macOS や Windows で生成した PNG を CI（ubuntu-latest）で比較すると必ず差分が出る。正しい手順は CI 環境に合わせた Linux 上で `--update-snapshots` を一度実行してコミットすることで、それは開発チームが意図的に行う必要がある。CI に `--update-snapshots` を常設すると毎回ベースラインが更新されリグレッションを検知できなくなるため採用しない。

### 生成ファイル 75 件を git から除外していない

orval が生成する `frontend/src/api/generated/` 以下の 75 ファイルは現在 git 管理されている。CI で `npm run generate && git diff --exit-code` のドリフトチェックを追加することは推奨するが、75 ファイルを `.gitignore` に移す作業は既存チェックアウトへの影響が大きく、専用 PR で実施する予定。

---

## まとめ

| 修正 | ファイル | 優先度 |
|------|----------|--------|
| `customFetch` エンベロープ返却 + 非 JSON フォールバック | `frontend/src/api/client.ts` | P1 |
| MSW `onUnhandledRequest: 'error'` | `frontend/src/test/setup.ts` | P2 |
| `@testing-library/jest-dom` 明示追加 | `frontend/package.json` | P3 |
| `webServer` ブロックのアンコメント | `frontend/playwright.config.ts` | P3 |
| CI PR に build ステップ追加 | `.github/workflows/ci-pr.yml` | P1 |
| CI Nightly の重複サーバー起動削除 | `.github/workflows/ci-nightly.yml` | P1 |

今回の修正で学んだ最大のポイントは「**ツール間の型契約とランタイムの振る舞いを合わせる**」重要性だ。orval が生成した型は `{ data, status, headers }` を期待しているにもかかわらず、`customFetch` が生の JSON を返していたことで型チェックをパスしながらランタイムは壊れていた。生成コードを導入するときは、その生成コードが期待するインターフェースをカスタムレイヤーが正確に実装しているかを必ず確認したい。
