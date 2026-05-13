# StorybookCreatorAgent - 自動ストーリー生成エージェントの実装

## 概要

TDD Todo App に新しいカスタム Copilot エージェント **StorybookCreatorAgent** を実装しました。このエージェントは、React コンポーネントから自動的に包括的な Storybook ストーリーを生成し、CSF 3.0 フォーマット、MSW API モッキング、アクセシビリティ優先設計を実現します。

---

## ストーリーブックが重要な理由

### 従来のアプローチの課題

React コンポーネント開発では、通常以下の成果物が作られます：

| 成果物 | 目的 | 対象者 |
|-------|------|------|
| **.test.tsx** | 動作検証・ロジックテスト | 開発者 |
| **.tsx** | コンポーネント実装 | 開発者 |
| ❌ **UI ドキュメント** | UI 状態の可視化 | デザイナー、QA、ステークホルダー |

### Storybook のメリット

```
✅ UI ドキュメント: コンポーネントの全状態を可視化
✅ デザイン検証: レスポンシブ、アクセシビリティ、色彩
✅ 開発効率化: コンポーネント単体での開発・テストが容易
✅ 品質保証: エラーハンドリング、エッジケースの確認
✅ ステークホルダー共有: コード読めない人も理解できる
```

### 手動で 95+ のストーリーを作るコスト

11 個のコンポーネント × 5〜11 ストーリーバリアント × 手動記述 = **大量の人手と時間**

```
想定工数:
- AppForm コンポーネント 1 個 = 11 ストーリーバリアント
  × 手動記述 1 バリアント あたり 15 分
  = 165 分 ≈ 3 時間

全 11 コンポーネント:
  165 分 × 11 = 1,815 分 ≈ 30 時間の手動作業 😰

StorybookCreatorAgent:
  → 自動生成で数分で完了 ⚡
```

---

## StorybookCreatorAgent の役割

### エージェントの職責

StorybookCreatorAgent は、以下のワークフローで自動的にストーリーを生成します：

```
1. コンポーネント実装ファイルを読む
   ↓
2. テストケースから期待される挙動を分析
   ↓
3. ストーリーバリアント（happy path、エラー、ローディングなど）を計画
   ↓
4. CSF 3.0 形式でストーリーコードを生成
   ↓
5. MSW ハンドラを設定（API 依存コンポーネント用）
   ↓
6. TypeScript 厳密型チェック
   ↓
7. Storybook ビルド検証
   ↓
8. Git コミット
```

### 入力例

```typescript
// ユーザーからの指示
task(
  name: 'storybook-creator',
  prompt: 'Generate Storybook stories for all components in frontend/src/features/ 
           that lack .stories.tsx files. Include happy paths, error states, 
           loading states, and edge cases. Use MSW for API mocking.',
  agent_type: 'StorybookCreatorAgent',
)
```

---

## CSF 3.0 フォーマット

### CSF 3.0 とは？

**Component Story Format 3.0** は Storybook 8+ の推奨フォーマットです。

### 特徴

#### 1. **簡潔なメタデータ**

```typescript
// 修正前（CSF 2.0）
export default {
  title: 'Components/Button',
  component: Button,
  args: {
    label: 'Click me',
  },
}

export const Default = (args) => <Button {...args} />

// 修正後（CSF 3.0）
type Story = StoryObj<typeof Button>

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
}

export default meta

export const Default: Story = {
  args: {
    label: 'Click me',
  },
}
```

#### 2. **完全な TypeScript 型チェック**

```typescript
// CSF 3.0: 型安全
type Story = StoryObj<typeof AppForm>

export const WithValidation: Story = {
  args: {
    defaultValue: 'Test',
    onSubmit: () => {},
    onCancel: () => {},
    isLoading: false,
    // TypeScript が型チェック：引数が Props に適合するか確認
  },
}
```

#### 3. **アクティブなストーリー検出**

```typescript
// 自動生成される Meta 情報
const meta: Meta<typeof TodoForm> = {
  title: 'Features/TodoForm',
  component: TodoForm,
  tags: ['autodocs'],  // 自動ドキュメント生成
}
```

---

## MSW 統合による API モッキング

### なぜ MSW が必要なのか

Storybook で API 依存コンポーネントをテストするとき、以下の課題があります：

```
❌ リアル API を呼び出す
  → テストデータを用意しなければならない
  → ネットワークレイテンシで開発が遅くなる
  → CI が不安定になる

✅ MSW でモック
  → 完全に制御されたテストデータ
  → 即座に応答
  → CI で安定
```

### StorybookCreatorAgent の MSW 設定

```typescript
// AppListPage.stories.tsx
export const Default: Story = {
  parameters: {
    msw: {
      handlers: [
        http.get('/api/v1/apps', () =>
          HttpResponse.json({
            success: true,
            data: [
              {
                id: 'app-1',
                name: 'My Todo App',
                createdAt: '2026-04-01T00:00:00Z',
                updatedAt: '2026-04-01T00:00:00Z',
              },
              {
                id: 'app-2',
                name: 'Project Management',
                createdAt: '2026-04-02T00:00:00Z',
                updatedAt: '2026-04-02T00:00:00Z',
              },
            ],
          }),
        ),
      ],
    },
  },
}

// ローディング状態: リクエストが完了しないハンドラ
export const LoadingState: Story = {
  parameters: {
    msw: {
      handlers: [
        http.get('/api/v1/apps', async () => {
          // Promise が解決されないため、UI はローディング状態を表示し続ける
          await new Promise(() => {})
          return HttpResponse.json({ success: true, data: [] })
        }),
      ],
    },
  },
}

// エラー状態：409 Conflict
export const ConflictError: Story = {
  parameters: {
    msw: {
      handlers: [
        http.post('/api/v1/apps', () =>
          HttpResponse.json(
            {
              success: false,
              data: null,
              error: {
                code: 'CONFLICT',
                message: 'App name already exists',
              },
            },
            { status: 409 },
          ),
        ),
      ],
    },
  },
}
```

### MSW がカバーする API エンドポイント

StorybookCreatorAgent は以下の 9 個の API エンドポイントをモック化します：

| エンドポイント | メソッド | 用途 | ストーリー |
|-------------|--------|------|---------|
| `/api/v1/apps` | GET | アプリ一覧取得 | AppListPage |
| `/api/v1/apps` | POST | アプリ新規作成 | AppCreatePage, AppForm |
| `/api/v1/apps/:appId` | GET | アプリ詳細取得 | AppDetailPage |
| `/api/v1/apps/:appId` | PUT | アプリ更新 | AppEditPage, AppForm |
| `/api/v1/apps/:appId` | DELETE | アプリ削除 | AppDetailPage |
| `/api/v1/apps/:appId/todos` | GET | Todo 一覧取得 | AppDetailPage |
| `/api/v1/apps/:appId/todos` | POST | Todo 新規作成 | TodoForm |
| `/api/v1/apps/:appId/todos/:todoId` | PUT | Todo 更新 | TodoForm |
| `/api/v1/apps/:appId/todos/:todoId` | DELETE | Todo 削除 | TodoItem |

---

## アクセシビリティ優先設計

### WCAG 2.1 に準拠したストーリー

StorybookCreatorAgent は、アクセシビリティをストーリー設計の最初から組み込みます：

#### 1. **フォーカス状態の可視化**

```typescript
export const FocusableButton: Story = {
  args: {
    label: 'Save',
  },
  parameters: {
    docs: {
      description: {
        story: 'Use Tab key to focus. Blue outline should be visible.',
      },
    },
  },
}
```

#### 2. **キーボードナビゲーション**

```typescript
export const KeyboardNavigation: Story = {
  args: { /* ... */ },
  decorators: [
    (Story) => (
      <div>
        <p>Try pressing Tab, Shift+Tab, Enter, Space</p>
        <Story />
      </div>
    ),
  ],
}
```

#### 3. **スクリーンリーダー対応**

```typescript
export const ScreenReaderFriendly: Story = {
  args: {
    // ARIA ラベルを含む UI
    ariaLabel: 'Delete todo item',
    ariaDescribedBy: 'delete-confirmation',
  },
}
```

#### 4. **色対比チェック**

```typescript
export const HighContrast: Story = {
  args: { /* ... */ },
  parameters: {
    // Storybook Accessibility Plugin で自動チェック
    a11y: {
      config: {
        rules: [
          { id: 'color-contrast', enabled: true },
          { id: 'valid-aria-role', enabled: true },
        ],
      },
    },
  },
}
```

---

## StorybookCreatorAgent のガバニングルール

エージェントの動作を厳密に規定するルール：

### 生成ルール

1. **CSF 3.0 のみ** - 古い CSF 2.0 フォーマットは使用しない
2. **型安全** - すべてのストーリー引数が完全に型付けされている（`any` なし）
3. **バリアント必須** - 最低限：Default, Error, Loading, Empty
4. **MSW 統合** - API 呼び出しのあるコンポーネントはすべて MSW でモック
5. **独立性** - ストーリー間で状態を共有しない

### 禁止事項

- ❌ 存在しないコンポーネントのストーリー作成
- ❌ リアル API への呼び出し
- ❌ 新しい npm パッケージの追加
- ❌ コンポーネント実装コードの変更
- ❌ `any` 型の使用

---

## 実装例：TodoForm ストーリー

完成した StorybookCreatorAgent から生成されたストーリーの例：

```typescript
import type { Meta, StoryObj } from '@storybook/react'
import { http, HttpResponse } from 'msw'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { TodoForm } from './TodoForm'

type Story = StoryObj<typeof TodoForm>

const mockTodo = {
  id: 'todo-1',
  title: 'Fix loading state bug',
  completed: false,
  createdAt: '2026-04-29T00:00:00Z',
  updatedAt: '2026-04-29T00:00:00Z',
  appId: 'app-1',
}

const meta: Meta<typeof TodoForm> = {
  title: 'Features/TodoForm',
  component: TodoForm,
  tags: ['autodocs'],
  decorators: [
    (Story) => {
      const queryClient = new QueryClient()
      return (
        <QueryClientProvider client={queryClient}>
          <Story />
        </QueryClientProvider>
      )
    },
  ],
}

export default meta

// 新規作成モード
export const CreateMode: Story = {
  args: {
    mode: 'create',
    appId: 'app-1',
    onCancel: () => console.log('Cancel'),
    onSuccess: () => console.log('Success'),
  },
  parameters: {
    msw: {
      handlers: [
        http.post('/api/v1/apps/app-1/todos', () =>
          HttpResponse.json({
            success: true,
            data: { id: 'todo-2', title: 'New Todo', completed: false },
          }),
        ),
      ],
    },
  },
}

// 編集モード
export const EditMode: Story = {
  args: {
    mode: 'edit',
    todo: mockTodo,
    appId: 'app-1',
    onCancel: () => console.log('Cancel'),
    onSuccess: () => console.log('Success'),
  },
  parameters: {
    msw: {
      handlers: [
        http.put('/api/v1/apps/app-1/todos/todo-1', () =>
          HttpResponse.json({
            success: true,
            data: { ...mockTodo, title: 'Updated Title' },
          }),
        ),
      ],
    },
  },
}

// ローディング中
export const LoadingState: Story = {
  args: {
    mode: 'create',
    appId: 'app-1',
    onCancel: () => {},
    onSuccess: () => {},
  },
  parameters: {
    msw: {
      handlers: [
        http.post('/api/v1/apps/app-1/todos', async () => {
          // 永遠に完了しない → ローディング UI を表示
          await new Promise(() => {})
          return HttpResponse.json({ success: true, data: {} })
        }),
      ],
    },
  },
}

// バリデーションエラー
export const ValidationError: Story = {
  args: {
    mode: 'create',
    appId: 'app-1',
    onCancel: () => {},
    onSuccess: () => {},
  },
}
```

---

## ビルド検証

StorybookCreatorAgent は自動的にストーリー生成後に Storybook ビルドを実行し、成功を確認します：

```bash
$ npm run build-storybook

✓ 541 modules transformed successfully
✓ Vite built in 1.32s
✓ Storybook build completed successfully
✓ Output: storybook-static/
```

---

## エージェントのコミット権限

StorybookCreatorAgent は Git コミット権限を持ち、ストーリー生成完了後に自動コミットします：

```bash
git add frontend/src/features/**/*.stories.tsx STORYBOOK_*.md
git commit -m "feat: add Storybook stories for 11 components with 95+ variants"
git commit --amend --no-edit -m "feat: add Storybook stories for 11 components with 95+ variants

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## ワークフロー統合

### TDD サイクルでの位置付け

```
Red       → 失敗テスト作成
  ↓
Green     → 実装コード作成
  ↓
Refactor  → コード改善
  ↓
Review    → コードレビュー
  ↓
✨ StorybookCreatorAgent ← 新しいステップ
         → ストーリー自動生成
         → デザイナー・QA 検証
         → ドキュメント完成
```

---

## 今後の拡張

StorybookCreatorAgent は以下の拡張に対応できるよう設計されています：

1. **ビジュアルリグレッションテスト統合**: Chromatic との連携
2. **パフォーマンス測定**: Storybook の performance addon との連携
3. **コンポーネント相互作用テスト**: Storybook interactions 機能
4. **デザインシステムドキュメント**: Design tokens の自動同期

---

## まとめ

StorybookCreatorAgent により以下が実現されました：

| 項目 | 価値 |
|-----|------|
| **自動化** | 30 時間の手動作業 → 数分で完了 |
| **品質** | CSF 3.0、型安全、MSW 統合、アクセシビリティ |
| **保守性** | ストーリー生成ルール の一元化 |
| **可視化** | 95+ ストーリーバリアント による UI ドキュメント |
| **効率** | デザイナー、QA、ステークホルダーとの協働 |

このエージェントは、React コンポーネント開発における**生産性と品質の両立**を実現します。

---

**実装日**: 2026-04-29  
**ステータス**: ✅ 本番対応完了  
**ストーリーファイル**: 11 個  
**ストーリーバリアント**: 95+
