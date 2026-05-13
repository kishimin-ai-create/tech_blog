# pushはユニットテストのみ・PRで統合テストも走らせるCI設計

## 対象読者

- GitHub Actions で TypeScript バックエンドの CI を組んでいる
- テストが増えてきて「毎回全テストを回すと遅い」と感じている
- テスト種別ごとに Vitest の設定を分けたい

## この記事が扱う範囲

- `backend-ci.yml` の push / PR 分岐設計
- `vitest.unit.config.ts` と `vitest.integration.config.ts` の分離方法
- `package.json` のスクリプト設計
- 以前の `backend.yaml` との比較

統合テストの内容設計（フォルダ構造・Shared Storage パターン等）は別記事で扱う。

---

## 背景：統合テストを追加したら「全部毎回回す」が辛くなった

プロジェクトには Vitest によるユニットテストが co-locate（ソースと同階層）で存在していた。今回、17 ファイルの統合テストスイートを追加した。

統合テストは実際の InMemory リポジトリや DI コンテナを使うため、モックを使うユニットテストより実行時間が長くなりやすい。これを毎 push で実行するとフィードバックサイクルが低下する。

一方で統合テストはレビュー前のコード品質チェックとして有効なので、**PR 時には必ず実行したい**。

この要件から「push 時はユニットテストのみ・PR 時に統合テストも追加で実行する」という CI 構成を設計した。

---

## 旧 CI との比較

**旧: `backend.yaml`**

```yaml
on: ["push", "pull_request"]

jobs:
  unit-test:
    steps:
      - name: Vitest unit test
        run: npm test  # ← すべてのテストを実行（区別なし）
```

すべてのテストを単一ジョブで実行していた。テスト種別の区別がなく、統合テストを追加すると自動的にすべての push で実行されてしまう。

**新: `backend-ci.yml`**

```yaml
on:
  push:
    branches: [main, master]
    paths:
      - 'backend/**'
  pull_request:
    branches: [main, master]
    paths:
      - 'backend/**'

jobs:
  unit:
    name: Lint, Typecheck & Unit Tests
    runs-on: ubuntu-latest
    steps:
      - name: Unit tests
        run: npm run test:unit   # ← ユニットのみ

  integration:
    name: Integration Tests
    if: github.event_name == 'pull_request'   # ← PR 時のみ
    runs-on: ubuntu-latest
    steps:
      - name: Integration tests
        run: npm run test:integration
```

`unit` ジョブは push / PR の両方で実行される。`integration` ジョブは `if: github.event_name == 'pull_request'` によって PR 時のみ実行される。

---

## Vitest 設定ファイルの分離

テスト種別を CI で分けるためには、Vitest がどのファイルを実行するかを制御する必要がある。設定ファイルを2つに分けることで対応した。

**`vitest.unit.config.ts`**（ユニットテスト用）

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    exclude: [
      '**/node_modules/**',
      '**/dist/**',
      'src/tests/integrations/**',   // ← 統合テストを除外
    ],
  },
});
```

デフォルトの `exclude` に `src/tests/integrations/**` を追加している。これにより `vitest.unit.config.ts` を指定した実行では統合テストが一切動かない。

**`vitest.integration.config.ts`**（統合テスト用）

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['src/tests/integrations/**/*.test.ts'],   // ← 統合テストだけを含める
  },
});
```

`include` を限定することで、ユニットテストは完全に除外される。

---

## package.json のスクリプト設計

```json
{
  "scripts": {
    "test": "vitest run --reporter=default --reporter=json --outputFile=test-result.json",
    "test:unit": "vitest run --config vitest.unit.config.ts --reporter=default",
    "test:integration": "vitest run --config vitest.integration.config.ts --reporter=default"
  }
}
```

| スクリプト | 用途 | 実行対象 |
|---|---|---|
| `npm test` | 全テスト（JSON 出力付き） | 旧来互換。全ファイル |
| `npm run test:unit` | CI push 時・ローカル高速確認 | ユニットテストのみ |
| `npm run test:integration` | CI PR 時・統合確認 | 統合テストのみ |

ローカル開発では `test:unit` を使い、PR を出す前に `test:integration` で最終確認するという使い方を想定している。

---

## CI ジョブの依存関係とパス絞り込み

```yaml
on:
  push:
    paths:
      - 'backend/**'
      - '.github/workflows/backend-ci.yml'
  pull_request:
    paths:
      - 'backend/**'
      - '.github/workflows/backend-ci.yml'
```

`paths` フィルタを設定しているため、frontend や docs の変更では backend CI が起動しない。モノレポ構成でのノイズを減らす工夫だ。

また、`unit` と `integration` は独立したジョブで、依存関係（`needs`）を持たせていない。ユニットテストが失敗しても統合テストジョブは並列で走る。この設計により「どのジョブが落ちたか」が明確になる。

---

## 設計上のトレードオフ

### push で統合テストを走らせない理由

統合テストは実際の依存を持つため実行が遅い傾向がある。push のたびに実行すると、小さなコミットに対するフィードバックが遅れる。開発中の高速な反復サイクルを守るため、push 時はユニットテストのみとした。

### PR で必ず統合テストを実行する理由

PR はレビュワーが見る前の最終チェックポイントだ。この時点で統合テストが落ちていれば、レビュー前にバグを検知できる。コードレビューの品質を守るために PR での統合テスト実行は外せない。

### 両方のジョブが `npm ci` を実行する

`unit` と `integration` は独立したジョブのため、それぞれ `npm ci` でインストールしている。`needs` を使ったインストール共有も選択肢だが、ジョブを分けることで並列実行と独立したログが得られるため、現状の構成を選択している。

---

## まとめ

- **Vitest の設定ファイルを `unit` と `integration` に分ける**ことで、実行対象を明確に制御できる
- **`if: github.event_name == 'pull_request'`** で統合テストジョブを PR 限定にする
- push 時はユニットテストで高速フィードバック、PR 時に統合テストで品質担保という役割分担が成立する
- `paths` フィルタでモノレポの不要なジョブ起動を防ぐ
- スクリプトを `test:unit` / `test:integration` に分けることでローカルでも同じ区別が使える
