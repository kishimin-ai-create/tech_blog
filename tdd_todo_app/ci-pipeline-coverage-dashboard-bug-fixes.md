# デプロイ後の CI の 3 つのバグが修正されました: ロック ファイル、ESLint クラッシュ、カバレッジ ダッシュボード

**タイプ:** 開発ブログ / リリースノート
**対象者:** このプロジェクトの CI パイプラインを維持または拡張するエンジニア
**コミット:** `a04d96a` → `d1e40bc` → `7432af2`

---

## 背景

GitHub Actions デプロイ ワークフロー (前のセッション) をマージした後、`ubuntu-latest` で実行された最初の CI により、Windows でのローカル開発では気づかれなかった 3 つの個別の障害が明らかになりました。それぞれのバグは独立していました (根本原因が異なり、ファイルが異なり、修正が異なりました)。しかし、パイプラインが確実に環境に優しいものになる前に、3 つすべてを解決する必要がありました。

この投稿では、何が失敗したか、なぜ失敗したか、何が変更されたかなど、各バグを文書化します。

---

## バグ 1 — Linux で `npm ci` が失敗する: esbuild プラットフォーム エントリが欠落している

**コミット:** `d1e40bc`
**ファイル:** `backend/package-lock.json`

### 症状

デプロイ ワークフローとテスト ワークフローの両方が、`npm ci` ステップですぐに失敗しました。

```
npm error Missing: esbuild@0.28.0 from lock file
npm error Missing: @esbuild/linux-x64@0.28.0
npm error Missing: @esbuild/linux-arm64@0.28.0
...
```

### 根本的な原因

`backend/package-lock.json` は Windows 上で生成されていました。ロック ファイルには、esbuild 用の Windows 固有のオプション プラットフォーム パッケージ (`@esbuild/win32-x64`、`@esbuild/win32-ia32` など) のみが含まれており、Linux に相当するものは含まれていません。

`npm ci` は設計上厳密です。ロック ファイルが現在のプラットフォームで予想されるすべてのオプションの依存関係を考慮していない場合、インストールを拒否します。 `ubuntu-latest` では、Linux プラットフォーム パッケージが予期されていましたが存在せず、致命的なエラーが発生しました。

これは一般的なクロスプラットフォーム トラップです。 `npm install` は、*現在の OS* に関連するオプション パッケージのみを記録するため、ある OS で生成されたロック ファイルが別の OS では不完全になることがよくあります。

### Fix

バックエンドで `npm install` を再実行し、すべてのプラットフォームの esbuild エントリを含むロック ファイルを再生成します。再生成されたファイルは 86 行増加し、19 行 (ネット +67) を削除し、既存の Windows エントリとともに `@esbuild/linux-x64`、`@esbuild/linux-arm64`、およびその他のプラットフォーム バイナリをピックアップしました。

```
backend/package-lock.json | 105 +++++++++++++++++++++++++++----
 1 file changed, 86 insertions(+), 19 deletions(-)
```

### 取り除く

開発マシンと CI ランナーが異なるオペレーティング システムを使用している場合は、コミットする前に、必ず Linux 環境で `package-lock.json` を再生成してください (または `npm install --os=linux --cpu=x64` クロスコンパイル フラグを使用してください)。あるいは、CI イメージと一致する Docker コンテナ内で `npm ci` をローカルで実行して、不一致を早期に検出します。

---

## バグ 2 — ESLint が `ecosystem.config.cjs` でクラッシュする

**コミット:** `a04d96a`
**ファイル:** `backend/eslint.config.mts`

### 症状

バックエンドの `npm run lint` が次のエラーで失敗しました:

```
Error: Error while loading rule '@typescript-eslint/await-thenable':
You have used a rule which requires type information, but don't have parserOptions set.
Occurred while linting: backend/ecosystem.config.cjs
```

### 根本的な原因

`eslint.config.mts` は、ファイル パターンの制限なしで、`tseslint.configs.recommendedTypeChecked` を最上位レベルでグローバルに適用します。

```ts
// backend/eslint.config.mts (before fix)
tseslint.configs.recommended,
tseslint.configs.recommendedTypeChecked,   // ← applied to every file
```

`recommendedTypeChecked` には、TypeScript コンパイラーの型チェック API を必要とする `@typescript-eslint/await-thenable` などのルールが含まれています。構成は `parserOptions.projectService` をセットアップしますが、そのブロックのスコープは `**/*.{ts,mts,cts}` ファイルのみです。

```ts
{
  files: ["**/*.{ts,mts,cts}"],
  languageOptions: {
    parserOptions: { projectService: true },
  },
},
```

`ecosystem.config.cjs` はプレーンな CommonJS ファイルです。TypeScript パーサーは構成されていません。 ESLint のフラット構成が、グローバルに適用される型認識ルールに基づいて ESLint を処理すると、処理する型情報がなく、致命的にクラッシュします。

### Fix

`"ecosystem.config.cjs"` を `globalIgnores` エントリに追加して、ESLint が処理しないようにします。

```diff
- ignores: ["coverage/**", "dist/**"],
+ ignores: ["coverage/**", "dist/**", "ecosystem.config.cjs"],
```

diff は 1 行です。その結果、ESLint は、TypeScript コンテキストを持たないファイルに型認識ルールを適用しようとするのではなく、ファイルを完全にスキップします。

### 取り除く

`tseslint.configs.recommendedTypeChecked` を使用する場合、スコープを TypeScript ファイルのみ (`files: ["**/*.ts"]`) にするか、非 TypeScript 構成ファイルを `globalIgnores` に追加します。 TS 解析可能なファイルに制限せずに型認識ルールをグローバルに適用すると、プロジェクト内のプレーンな JS/CJS ファイルでクラッシュします。

---

## バグ 3 — カバレッジ ダッシュボードに常に 0% が表示される

**コミット:** `7432af2`
**ファイル:** `scripts/generate-coverage-report.js`、`docs/coverage/index.html`

このコミットにより、同じスクリプト内の 2 つの異なるサブバグが修正されました。

### パート A — 間違った JSON 構造が想定されている (0% 問題)

#### 症状

生成された `docs/coverage/index.html` は、テストが実際のカバレッジ データで合格した場合でも、すべてのレイヤーのすべてのメトリック (行、分岐、関数、ステートメント) で 0% を示しました。

#### 根本的な原因

`coverage.total` からの `extractMetrics()` 読み取り:

```js
// Before fix
const total = coverage.total || {};
return {
  lines:      Math.round(total.lines?.pct     || 0),
  branches:   Math.round(total.branches?.pct  || 0),
  functions:  Math.round(total.functions?.pct || 0),
  statements: Math.round(total.statements?.pct|| 0)
};
```

これは、入力が `coverage-summary.json` であることを前提としています。これには、事前に集計されたパーセンテージを持つ `total` キーが*存在します*。しかし、Vitest/Istanbul は実際には `coverage-final.json` (**ファイル カバレッジ マップ**) を生成します。この形式では、すべての最上位キーは絶対ファイル パスです。 `total` はありません:

```json
{
  "/abs/path/to/src/app.ts": {
    "s": { "0": 5, "1": 3 },
    "b": { "0": [2, 1], "1": [0, 4] },
    "f": { "0": 4, "1": 0 }
  },
  "/abs/path/to/src/index.ts": { ... }
}
```

`coverage.total` は常に `undefined` であるため、すべての `?.pct` は `undefined` に短絡し、`|| 0` は静かにゼロに戻ります。エラーは決してスローされません。ダッシュボードではすべてゼロが表示されるだけです。

#### Fix

`extractMetrics()` は、すべてのファイル エントリを反復し、生の `s` (ステートメント)、`b` (ブランチ)、および `f` (関数) フィールドからヒット カウントを手動で集計するように書き直されました。

```js
function extractMetrics(coverage) {
  if (!coverage) return { lines: 0, branches: 0, functions: 0, statements: 0 };

  let stmtCovered = 0,   stmtTotal = 0;
  let branchCovered = 0, branchTotal = 0;
  let fnCovered = 0,     fnTotal = 0;

  for (const fileData of Object.values(coverage)) {
    // Statements: { "0": hitCount, "1": hitCount, ... }
    const stmts = fileData.s || {};
    for (const count of Object.values(stmts)) {
      stmtTotal++;
      if (count > 0) stmtCovered++;
    }

    // Branches: { "0": [taken, not_taken], "1": [...] }
    const branches = fileData.b || {};
    for (const counts of Object.values(branches)) {
      for (const count of counts) {
        branchTotal++;
        if (count > 0) branchCovered++;
      }
    }

    // Functions: { "0": hitCount, "1": hitCount, ... }
    const fns = fileData.f || {};
    for (const count of Object.values(fns)) {
      fnTotal++;
      if (count > 0) fnCovered++;
    }
  }

  const pct = (covered, total) =>
    total === 0 ? 0 : Math.round((covered / total) * 100);

  return {
    lines:      pct(stmtCovered, stmtTotal),   // Istanbul uses statements as proxy for lines
    branches:   pct(branchCovered, branchTotal),
    functions:  pct(fnCovered, fnTotal),
    statements: pct(stmtCovered, stmtTotal),
  };
}
```

主要な集計ルール:
- **ステートメント / 行** — `s` は、ステートメント ID とヒット数のマップです。 `count > 0` でエントリをカウントする
- **ブランチ** — `b` は、アーム数の配列へのブランチ ID のマップです (例: `[taken, not_taken]`)。 `count > 0` を使用して個々のアームをカウントする
- **関数** — `f` は、関数 ID とヒット数のマップです。 `count > 0` でエントリをカウントする

### パート B — 404 につながる「レポートの表示」リンク

#### 症状

ダッシュボードの [レポートの表示] リンクをクリックすると、404 ページが返されました。

#### 根本的な原因

ダッシュボードは、`docs/coverage/index.html` (リポジトリ ルートの 2 つ下のディレクトリ レベル) に書き込まれます。リンクでは、単一ドットからなる相対パスが使用されています。

```html
<!-- Before fix -->
<a href="../frontend/coverage/index.html">View Report</a>
<a href="../backend/coverage/unit/index.html">View Report</a>
```

`docs/coverage/` から、`../frontend/` は `docs/frontend/coverage/index.html` に解決されますが、これは存在しません。実際のレポート ファイルは `frontend/coverage/index.html` (リポジトリ ルート レベル) に存在するため、2 つ上のレベルが必要です。

#### Fix

すべてのレポート `href` を `../X/` から `../../X/` に変更しました。

```diff
- <td><a href="../frontend/coverage/index.html" target="_blank">View Report</a></td>
+ <td><a href="../../frontend/coverage/index.html" target="_blank">View Report</a></td>

- <td><a href="../backend/coverage/unit/index.html" target="_blank">View Report</a></td>
+ <td><a href="../../backend/coverage/unit/index.html" target="_blank">View Report</a></td>
```

### 取り除く

Istanbul / Vitest は、形状の異なる 2 つの異なる JSON 出力ファイルを生成します。

|ファイル |トップレベルのキー | `total`はありますか？ |使用例 |
|---|---|---|---|
| `coverage-summary.json` | `total`、`<file-path>`、… | ✅ はい |簡単な合計、簡単な読み取り |
| `coverage-final.json` | `<file-path>`、… (のみ) | ❌ いいえ |ファイルごとの詳細、分岐/ステートメント データ |

集計パーセンテージのみが必要な場合は、`coverage-summary.json` を読んでください (`total.lines.pct` などがあります)。 `coverage-final.json` を消費する場合は、上記のように `s`、`b`、および `f` を自分で集計する必要があります。

生成された HTML 内の相対パス リンクの場合は、出力ファイルを生成するスクリプトではなく、常に *出力ファイル* のディレクトリの深さをカウントします。

---

## まとめ

| # |ファイルが変更されました |根本原因 |修正 |
|---|---|---|---|
| 1 | `backend/package-lock.json` | Windows 上で生成されたロック ファイル。 Linux esbuild プラットフォーム パッケージが見つかりません | `npm install` で再生成され、すべてのプラットフォームのエントリが生成されます。
| 2 | `backend/eslint.config.mts` | `recommendedTypeChecked` はグローバルに適用されます。タイプパーサーのない CJS ファイルでクラッシュしました。 `ecosystem.config.cjs` を `globalIgnores` に追加 |
| 3a | `scripts/generate-coverage-report.js` |ファイルマップ JSON から存在しない `coverage.total` を読み取ります。常に 0% | `extractMetrics` を書き換えて、すべてのファイル エントリから `s`/`b`/`f` を集約しました。
| 3b | `scripts/generate-coverage-report.js` | `href` の深さが 1 レベル不足していたことを報告します。 `frontend/` ではなく `docs/frontend/` にリンク |すべてのレポートの href を `../X/` → `../../X/` に変更しました。

3 つのバグはすべて、Windows でのローカル開発中には目に見えず、`ubuntu-latest` CI ランナー上で、またはブラウザーでダッシュボードを開いたときにのみ表面化しました。これは、クロスプラットフォームのロック ファイル、グローバル スコープの lint ルール、および生成された出力内の相対パスはすべて、CI ワークフローをマージする前に明示的な検証に値することを思い出させます。
