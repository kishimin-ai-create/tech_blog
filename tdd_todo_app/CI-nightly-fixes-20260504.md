# CI 夜間テスト: ESLint と Playwright セレクタの問題解決

**対象読者**: TDD と CI/CD パイプラインに取り組むエンジニア  
**レポジトリ**: Kazuma-Ishimine/TDD_todo_app  
**コミット**: c3693df - fix: ensure all CI checks pass  
**実行日**: 2026 年 5 月 4 日

## 概要

夜間 CI パイプライン（`ci-nightly.yml`）で実行される全テストを成功させるため、次の 2 つのクラスの問題を特定し修正しました：

1. **バックエンド ESLint no-console 警告** (2 ファイル)
2. **フロントエンド Playwright E2E テストセレクタの陳腐化** (1 ファイル)

修正後、フロントエンド 147 テスト + バックエンド 391 テスト + Playwright E2E テスト全て ✓ 成功

---

## 問題 1: ESLint no-console 警告の過度な厳格性

### 背景

ESLint の `no-console` ルール は、開発中のデバッグ出力をコードに残さないようにするためのルールです。ただし、サーバアプリケーションやデータベース移行スクリプトでは、`console.log()` や `console.error()` が**システムの診断出力として必須**となるため、すべての出力を禁止するのは過度です。

### 該当ファイル

#### 1. `backend/src/server.ts`

```javascript
/* eslint-disable no-console */

import { serve } from '@hono/node-server';
import honoApp from './index';

const port = Number(process.env.PORT ?? 3000);

serve({
  fetch: honoApp.fetch,
  port,
});

console.log(`Server running at http://localhost:${port}`);
```

**理由**: サーバ起動メッセージ「`Server running at http://localhost:3000`」は、アプリケーションが正常に起動したかどうかを確認するための必須の診断出力です。この出力がなければ、開発者はサーバが起動したかどうか判断できません。

#### 2. `backend/src/infrastructure/migrate.ts`

```javascript
/* eslint-disable no-console */

// ... (import 省略)

async function migrate(): Promise<void> {
  // ...
  for (const file of sqlFiles) {
    const [rows] = await connection.execute<MigrationRow[]>(
      'SELECT name FROM _migrations WHERE name = ?',
      [file],
    );

    if (rows.length > 0) {
      console.log(`  skip   ${file}`);
      continue;
    }

    const sql = await readFile(join(migrationsDir, file), 'utf-8');
    await connection.query(sql);
    await connection.execute(
      'INSERT INTO _migrations (name) VALUES (?)',
      [file],
    );
    console.log(`  apply  ${file}`);
  }

  console.log('Migration complete.');
}

migrate().catch((err: unknown) => {
  console.error('Migration failed:', err);
  process.exit(1);
});
```

**理由**: マイグレーション処理では、実行状況（スキップ、適用、完了）やエラーをログ出力することが、運用上の確認と診断に不可欠です。どのマイグレーションが実行されたか、失敗したかを把握する手段がなくなってしまいます。

### 解決方法

両ファイルの先頭に `/* eslint-disable no-console */` を追加しました：

```javascript
/* eslint-disable no-console */

// ... ファイルの残り
```

### ベストプラクティス

ESLint の no-console ルールを使う際は、以下を検討してください：

| ユースケース | ルール適用 | 理由 |
|---|---|---|
| Web アプリケーション（ブラウザ） | 有効 | コンソール出力は開発者ツールに隠れ、本番環境では不要 |
| Node.js サーバアプリケーション | **除外** | サーバ起動、ヘルスチェック、エラー報告は diagnostics に必須 |
| CLI ツール | **除外** | コマンドライン出力がツールの主要な機能 |
| ユーティリティ・ライブラリ | 有効 | 内部ロギングはより構造化した logging library に移譲 |
| マイグレーション・スクリプト | **除外** | 実行状況と診断情報を確認する主要な方法 |

---

## 問題 2: Playwright E2E テストセレクタの陳腐化

### 背景

フロントエンド TodoItem コンポーネントのスタイリングが更新され、CSS クラスが変更されました。Playwright の E2E テストで使用していた CSS クラスセレクタが古い値のままだったため、テストが失敗していました。

### 原因の特定

**変更前**:
```typescript
// frontend/e2e/crud.spec.ts (古いセレクタ)
const todoItem = page.locator('div.p-3.border.rounded.bg-white').filter({ hasText: 'Write CRUD E2E test' });
```

**TodoItem コンポーネントの現在の実装**:
```typescript
// frontend/src/features/app-detail/components/TodoItem.tsx
export function TodoItem({ todo, appId, onRefresh }: Props) {
  return (
    <div className="p-4 border border-gray-200 rounded-lg bg-white shadow-sm">
      {/* ... */}
    </div>
  )
}
```

**CSS クラスの変更**:
- `p-3` → `p-4` (パディングが小さいから大きいに)
- `border` → `border border-gray-200` (境界線色を明示的に灰色に)
- `rounded` → `rounded-lg` (角丸が小さいから大きいに)
- （追加）`shadow-sm` (淡い影を追加)

### 修正内容

`frontend/e2e/crud.spec.ts` の 2 箇所のセレクタを更新：

```typescript
// 更新前
const todoItem = page.locator('div.p-3.border.rounded.bg-white').filter({ hasText: 'Write CRUD E2E test' });

// 更新後
const todoItem = page.locator('div.p-4.border.border-gray-200.rounded-lg.bg-white').filter({ hasText: 'Write CRUD E2E test' });
```

同様に 2 番目の `updatedTodoItem` セレクタも同じパターンで更新。

### セレクタ戦略のベストプラクティス

CSS クラスベースのセレクタは保守性が低い傾向があります。以下の代替案を検討してください：

| 戦略 | メリット | デメリット | 推奨度 |
|---|---|---|---|
| CSS クラスセレクタ | シンプル、直感的 | スタイル変更で陳腐化 | ⭐⭐ |
| **data-testid 属性** | **スタイル変更に強い、意図が明確** | **テストロジックを本番コードに混在** | **⭐⭐⭐⭐⭐** |
| Role / Label / Text | セマンティック、ユーザー視点 | 複雑な場合は難しい | ⭐⭐⭐⭐ |
| 複合セレクタ | より安定 | 複雑、メンテナンスが増える | ⭐⭐⭐ |

**推奨改善案**：

```typescript
// TodoItem.tsx に data-testid を追加
<div 
  className="p-4 border border-gray-200 rounded-lg bg-white shadow-sm" 
  data-testid="todo-item"
>
  {/* ... */}
</div>

// テストでは data-testid を使用
const todoItem = page.getByTestId('todo-item').filter({ hasText: 'Write CRUD E2E test' });
```

これにより、CSS 変更時にもセレクタは影響を受けません。

---

## CI 夜間パイプラインの全体像

修正後、CI が実行するテスト範囲は以下の通りです：

```yaml
# .github/workflows/ci-nightly.yml
jobs:
  e2e-full:
    - Frontend Lint & Typecheck ✓
    - Frontend 147 Unit Tests ✓
    - Frontend Production Build ✓
    - Storybook Build ✓
    - Playwright E2E Tests (全ブラウザ, @visual 除外) ✓
    - Visual Regression Tests (Chromium 限定, @visual のみ) ✓

# バックエンドも並行実行
    - Backend Lint & Typecheck ✓
    - Backend 391 Unit Tests ✓
```

---

## 実装の確認

修正を適用した後、以下の検証結果が得られました：

✅ **フロントエンド**
- Lint: エラーなし
- Typecheck: エラーなし
- Unit Tests: 147/147 成功
- Production Build: 成功
- Storybook Build: 成功
- Playwright E2E: 6 テスト全て成功

✅ **バックエンド**
- Lint: エラーなし (no-console 警告を正当に除外)
- Typecheck: エラーなし
- Unit Tests: 391/391 成功

---

## 学習ポイント

### 1. ESLint ルールの文脈依存性

グローバルな linting ルールは強力ですが、コンテキストに応じて例外を認識する必要があります。「すべての `console.log()` は悪い」という一般化は、CLI やサーバアプリケーションでは成り立ちません。

**対応**:
- ファイルレベルで無効化: `/* eslint-disable rule-name */`
- ルール設定で環境別に変更: `overrides` セクションで Node.js 環境は no-console を緩和
- または logging library（Winston, Pino など）の導入を検討

### 2. E2E テストのセレクタ脆弱性

UI フレームワークやデザイン更新のたびに E2E テストセレクタが陳腐化するのは、保守コストの大きな課題です。

**対応**:
- `data-testid` 属性を本番コードに追加（テスト観点を組み込む）
- セレクタ戦略をドキュメント化し、レビュー時に確認
- 定期的なメンテナンス サイクルを E2E テストに組み入れる

### 3. 段階的な自動化テスト

TDD_todo_app は以下の層を組み合わせています：

1. **Unit Tests** (147 個) → コンポーネント・関数の単位テスト
2. **Visual Regression** (@visual) → スナップショットテスト
3. **E2E Tests** (6 個) → ユーザーフローの統合テスト

それぞれが異なる脆弱性を検出するため、すべてが必要です。

---

## まとめ

| 項目 | 内容 |
|---|---|
| **修正内容** | ESLint no-console 警告の抑制 × 2 + Playwright セレクタ更新 × 2 |
| **所要時間** | エビデンス収集と修正適用 |
| **結果** | フロントエンド 147 + バックエンド 391 + Playwright E2E 全テスト成功 |
| **教訓** | Linting ルールと E2E セレクタは定期的なレビュー対象になるべき |

次回の CI 改善では、以下を検討します：

- [ ] `data-testid` を TodoItem などのテスト対象コンポーネントに追加
- [ ] E2E テストセレクタ戦略をドキュメント化
- [ ] ESLint の no-console ルールを環境別に設定（Node.js は除外）
