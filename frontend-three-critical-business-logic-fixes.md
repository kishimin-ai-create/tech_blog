# フロントエンド3つの重要なビジネスロジック修正 — API応答ステータス検証の導入

## 対象読者

- React / TypeScript でビジネスロジックのテストと実装を行うエンジニア
- API設計と フロントエンド統合の責任を持つチームメンバー
- UI状態管理とエラーハンドリングパターンを学びたい初心者から中級者
- TDD (テスト駆動開発) で開発している組織

## この記事について

この記事では、TDD Todo App のフロントエンド React コンポーネントで見つかった **3つの重要なビジネスロジック欠陥** と、その修正方法を解説します。

### カバー範囲
- API 応答のステータスコード検証が必要な背景
- 各コンポーネントでの具体的な実装パターン
- 修正前後の動作の違い
- テストコードで検証する方法

### カバー外
- API設計の詳細な議論
- バックエンド実装の改善
- CSS やUIアニメーション

---

## 問題の根本原因

TDD Todo App では API ラッパーが **エラーステータス時に例外をスロー しない** 設計になっています。代わりに、すべてのレスポンスに対して `{ data, status }` 形式でレスポンスオブジェクトを返します。

**この設計の意図：**
- エラー処理を呼び出し側で制御できる柔軟性
- 非同期フローでの例外スローの複雑性を避ける

**予期しない副作用：**
コンポーネント側で `status` フィールドを **チェックしないと**、4xx/5xx のエラーレスポンスであっても成功時と同じロジックが実行されてしまいます。

### 修正前の動作フロー（問題あり）

```
API呼び出し
    ↓
[成功(200)] or [エラー(4xx/5xx)]
    ↓
常に onSuccess コールバック実行 ← ❌ ステータス確認なし
    ↓
UI 状態が"成功"に切り替わる（エラーなのに）
```

### 修正後の動作フロー（正常）

```
API呼び出し
    ↓
[成功(200-299)] → onSuccess コールバック実行 ✓
[エラー(4xx/5xx)] → コールバック実行 なし ✓
    ↓
UI 状態が正しく反映される
```

---

## 修正 1: TodoForm — 作成・編集時のステータス検証

### 問題の症状

- ユーザーが Todo のタイトルを入力して保存
- バックエンドが検証エラー (422) またはサーバーエラー (500) を返す
- **にもかかわらず、フォームが閉じてしまう**
- エラーメッセージが表示されず、ユーザーは何が起きたかわからない
- 親画面の更新が実行され、変更が反映されていないようにみえる

### 根本原因

`onSubmit` ハンドラーが `mutateAsync()` の戻り値をチェックしていません：

```typescript
// ❌ 修正前の問題あるコード
const onSubmit = async (values: FormValues) => {
  if (mode === 'edit' && todo) {
    await updateMutation.mutateAsync({ appId, todoId: todo.id, data: { title: values.title } })
  } else {
    await createMutation.mutateAsync({ appId, data: { title: values.title } })
  }
  onSuccess()  // ← ステータス確認なく常に実行される
}
```

### 修正方法

応答を受け取り、ステータスコードが 2xx (200-299) 範囲にあるかチェックしてから `onSuccess()` を呼び出します：

```typescript
// ✓ 修正後のコード
const onSubmit = async (values: FormValues) => {
  if (mode === 'edit' && todo) {
    const result = await updateMutation.mutateAsync({ 
      appId, 
      todoId: todo.id, 
      data: { title: values.title } 
    }) as unknown
    const typedResult = result as { status?: number }
    if (typedResult?.status && typedResult.status >= 200 && typedResult.status < 300) {
      onSuccess()  // ✓ 2xx の時だけ実行
    }
  } else {
    const result = await createMutation.mutateAsync({ 
      appId, 
      data: { title: values.title } 
    }) as unknown
    const typedResult = result as { status?: number }
    if (typedResult?.status && typedResult.status >= 200 && typedResult.status < 300) {
      onSuccess()  // ✓ 2xx の時だけ実行
    }
  }
}
```

### ステータス検証パターンの説明

1. **`as unknown` でキャスト**  
   TanStack Query の戻り値型が完全に定義されていないため、ステップバイステップで型安全性を確保します

2. **`as { status?: number }` で構造を仮定**  
   API ラッパーが返すオブジェクトに `status` フィールドが存在することを明示します

3. **2xx 範囲チェック**  
   ```typescript
   typedResult.status >= 200 && typedResult.status < 300
   ```
   HTTP 成功ステータスコードは 200-299 範囲です

### テストコード例

修正を検証するテストケースを 3 つ追加しました：

```typescript
describe('Error Cases - Failed PUT (4xx/5xx)', () => {
  it('when PUT returns 422 (validation error), then onSuccess is NOT called', async () => {
    const onSuccess = vi.fn()
    server.use(
      http.put('/api/v1/apps/app-1/todos/todo-1', () =>
        HttpResponse.json(
          { success: false, data: null, error: { code: 'VALIDATION_ERROR' } },
          { status: 422 },
        ),
      ),
    )
    renderWithProviders(
      <TodoForm
        mode="edit"
        todo={mockTodo}
        appId="app-1"
        onCancel={vi.fn()}
        onSuccess={onSuccess}
      />,
    )

    await user.clear(screen.getByRole('textbox', { name: /title/i }))
    await user.type(screen.getByRole('textbox', { name: /title/i }), 'Updated Todo')
    await user.click(screen.getByRole('button', { name: /save/i }))

    await new Promise(resolve => setTimeout(resolve, 100))
    expect(onSuccess).not.toHaveBeenCalled()  // ✓ 呼ばれない
  })

  it('when POST returns 409 (conflict), then onSuccess is NOT called', async () => {
    // 同様のテスト（作成モード）
  })

  it('when PUT returns 500, then onSuccess is NOT called', async () => {
    // 同様のテスト（サーバーエラー）
  })
})
```

### ユーザー体験への影響

- **修正前**：フォームが不可解に閉じる → ユーザーが保存できたと勘違い
- **修正後**：フォームが開いたまま → ユーザーが再編集を試みられる（エラーメッセージが別途表示される場合）

---

## 修正 2: TodoItem — 削除時のステータス検証

### 問題の症状

- ユーザーが Todo 削除ボタンをクリックして確認
- バックエンドが 404 (見つからない) または 500 (サーバーエラー) を返す
- **削除成功メッセージが表示される**（実際には失敗）
- **親画面が更新される**
- 実装は削除されていないのに、UI 上は削除されたように見える

### 根本原因

`handleDelete` メソッドが API 応答のステータスをチェックしていません：

```typescript
// ❌ 修正前の問題あるコード
const handleDelete = async () => {
  await deleteMutation.mutateAsync({ appId, todoId: todo.id })
  setSuccessMsg('Todo deleted successfully')  // ← ステータス確認なく常に実行
  setShowConfirm(false)
  onRefresh()
}
```

### 修正方法

削除応答を受け取り、ステータスコードが 2xx 範囲にある場合のみ UI を更新します：

```typescript
// ✓ 修正後のコード
const handleDelete = async () => {
  const result = await deleteMutation.mutateAsync({ appId, todoId: todo.id }) as unknown
  const typedResult = result as { status?: number }
  if (typedResult?.status && typedResult.status >= 200 && typedResult.status < 300) {
    setSuccessMsg('Todo deleted successfully')  // ✓ 2xx の時だけ実行
    setShowConfirm(false)
    onRefresh()
  }
}
```

### 修正のポイント

1. **成功メッセージの条件付き実行**  
   ステータスが 2xx のときのみ `setSuccessMsg()` を呼びます

2. **確認ダイアログのクローズを条件付けにする**  
   ユーザーが再試行できるようにダイアログを開いたままにします

3. **親画面の更新も条件付け**  
   `onRefresh()` は成功時のみ呼び出します

### テストコード例

```typescript
describe('Error Cases - Failed Delete (4xx/5xx)', () => {
  it('when DELETE returns 404, then does NOT show success message', async () => {
    server.use(
      http.delete('/api/v1/apps/app-1/todos/todo-1', () =>
        HttpResponse.json(
          { success: false, data: null, error: { code: 'NOT_FOUND' } },
          { status: 404 },
        ),
      ),
    )
    renderWithProviders(
      <TodoItem todo={mockTodo} appId="app-1" onRefresh={vi.fn()} />,
    )

    await user.click(screen.getByRole('button', { name: /delete/i }))
    await user.click(screen.getByRole('button', { name: /confirm/i }))

    await new Promise(resolve => setTimeout(resolve, 100))
    expect(screen.queryByText(/todo deleted successfully/i)).not.toBeInTheDocument()
  })

  it('when DELETE returns 404, then does NOT call onRefresh', async () => {
    const onRefresh = vi.fn()
    server.use(
      http.delete('/api/v1/apps/app-1/todos/todo-1', () =>
        HttpResponse.json(
          { success: false, data: null, error: { code: 'NOT_FOUND' } },
          { status: 404 },
        ),
      ),
    )
    renderWithProviders(
      <TodoItem todo={mockTodo} appId="app-1" onRefresh={onRefresh} />,
    )

    await user.click(screen.getByRole('button', { name: /delete/i }))
    await user.click(screen.getByRole('button', { name: /confirm/i }))

    await new Promise(resolve => setTimeout(resolve, 100))
    expect(onRefresh).not.toHaveBeenCalled()
  })

  it('when DELETE returns 500, then does NOT show success message', async () => {
    // サーバーエラーのテスト
  })

  it('when DELETE returns 422, then does NOT call onRefresh', async () => {
    // 検証エラーのテスト
  })
})
```

### ユーザー体験への影響

- **修正前**：削除に失敗したのに成功メッセージが表示 → データが実は残っている混乱
- **修正後**：ダイアログが閉じず、ユーザーが再試行できる → エラー状態が明確

---

## 修正 3: AppEditPage — ローディング状態とエラー状態の区別

### 問題の症状

- ユーザーが App 編集ページを開く（URL: `/apps/app-1/edit`）
- バックエンドが 404 Not Found または 500 Server Error を返す
- **ページが無限にローディング状態を表示**
- ユーザーはアプリケーションが動作停止したと思う
- エラーメッセージが一切表示されない

### 根本原因

コンポーネントが **2つの異なるエラー状態を区別していません**：

1. **ローディング中**：クエリが実行中（`isLoading=true`）
2. **エラー状態**：クエリ完了後、`app` が undefined（`isLoading=false, app=undefined`）

修正前のコードは `app` の有無だけで判定し、クエリが完了したかを確認していません：

```typescript
// ❌ 修正前の問題あるコード
const { data } = useGetApiV1AppsByAppId(appId)  // isLoading を取得していない
const app = (data as unknown as { data?: { data?: unknown } } | null)?.data?.data

if (!app) {
  return <div role="status">Loading...</div>  // ❌ エラーでも常に Loading
}
```

**問題：**
- クエリが 404 エラーで完了 → `data` は null または undefined
- コンポーネントは `!app` が true なので `Loading...` を表示
- ユーザーは永遠にローディング画面を見ることになる

### 修正方法

`useGetApiV1AppsByAppId` から `isLoading` も取得し、2つのケースを明確に分ける：

```typescript
// ✓ 修正後のコード
const { data, isLoading } = useGetApiV1AppsByAppId(appId)  // ✓ isLoading を追加

const app = (data as unknown as { data?: { data?: unknown } } | null)?.data?.data

if (isHidden) return null

if (isLoading) {  // ✓ クエリ実行中
  return <div role="status">Loading...</div>
}

if (!app && !isLoading) {  // ✓ クエリ完了後、app がない = エラー
  return (
    <div role="alert" className="p-4 bg-red-50 border border-red-300 rounded text-red-700">
      App not found. Please check the app ID.
    </div>
  )
}
```

### 状態遷移ダイアグラム

```
クエリ開始
    ↓
isLoading = true
    ↓
[成功] → data.data.data を解析 → app = <object> → フォーム表示 ✓
[失敗] → data = null or undefined
    ↓
isLoading = false, app = undefined
    ↓
エラー画面を表示 ✓
```

### テストコード例

```typescript
describe('Error Cases - GET Returns 404', () => {
  it('when GET /api/v1/apps/:id returns 404, then shows error message instead of loading', async () => {
    server.use(
      http.get('/api/v1/apps/app-1', () =>
        HttpResponse.json(
          { success: false, data: null, error: { code: 'NOT_FOUND' } },
          { status: 404 },
        ),
      ),
    )

    renderWithProviders(<AppEditPage appId="app-1" />)

    expect(await screen.findByRole('alert')).toBeInTheDocument()
    expect(screen.getByText(/app not found/i)).toBeInTheDocument()
  })

  it('when GET /api/v1/apps/:id returns 500, then shows error message instead of loading', async () => {
    server.use(
      http.get('/api/v1/apps/app-1', () =>
        HttpResponse.json(
          { success: false, data: null, error: { code: 'SERVER_ERROR' } },
          { status: 500 },
        ),
      ),
    )

    renderWithProviders(<AppEditPage appId="app-1" />)

    expect(await screen.findByRole('alert')).toBeInTheDocument()
    expect(screen.getByText(/app not found/i)).toBeInTheDocument()
  })

  it('when GET /api/v1/apps/:id returns 403, then shows error message instead of loading', async () => {
    // アクセス権限がないケース
  })
})
```

### ユーザー体験への影響

- **修正前**：404/500 エラーで無限ローディング → 修復不可能な状態
- **修正後**：エラーメッセージを表示 → ユーザーが URL を修正・再試行できる

---

## ステータス検証の一般化パターン

3つの修正から抽出できる **ステータス検証の標準パターン** を示します。

### パターン 1: ミューテーション（POST/PUT/DELETE）

```typescript
const result = await mutation.mutateAsync(payload) as unknown
const typedResult = result as { status?: number }

if (typedResult?.status && typedResult.status >= 200 && typedResult.status < 300) {
  // ✓ 成功時のロジック
  onSuccess()
  refresh()
  setSuccessMsg('操作に成功しました')
} else {
  // (オプション) エラー時のロジック
  setErrorMsg('操作に失敗しました')
}
```

### パターン 2: クエリ（GET）の状態管理

```typescript
const { data, isLoading, error } = useGetQuery(id)

if (isLoading) {
  return <LoadingSpinner />  // ✓ クエリ実行中
}

if (!data && !isLoading) {
  return <ErrorAlert message="リソースが見つかりません" />  // ✓ クエリ完了、データなし
}

if (error) {
  return <ErrorAlert message={error.message} />  // ✓ 明示的エラー
}

// ✓ ここまで到達 = data が確実に存在
return <DataDisplay data={data} />
```

### なぜ AS による型キャストなのか

`as unknown` を 2段階で行うのは、TypeScript の型安全性を保ちつつ、不完全な型定義に対応するためです：

```typescript
// ✓ 安全な型キャスト方法
const result = mutation.mutateAsync(...) as unknown
const typedResult = result as { status?: number }

// ❌ 危険（型チェッカーをバイパス）
const typedResult = (result as any).status

// ❌ 型エラーで即失敗（柔軟性がない）
const typedResult = result as { status: number }  // statusが必須と見なされ、undefined の可能性で型エラー
```

---

## テスト戦略

3つの修正では、合計 **10 個の新しいテストケース** を追加しました（118 → 128 テスト）。

### テストの構成

```
修正前：
- 成功シナリオのみ検証 (3 ケース)

修正後：
- 成功シナリオ (3 ケース) 
+ エラーシナリオ (10 ケース)
  = 合計 13 ケース
```

### エラーシナリオでカバーする HTTP ステータス

| 修正対象 | 404 | 422 | 409 | 500 |
|---------|-----|-----|-----|-----|
| TodoForm (PUT/POST) | - | ✓ | ✓ | ✓ |
| TodoItem (DELETE) | ✓ | ✓ | - | ✓ |
| AppEditPage (GET) | ✓ | - | - | ✓ |

### 各テストが検証する項目

**TodoForm テスト**
- ✓ エラーステータスで `onSuccess` が呼ばれない
- ✓ 作成・編集の両モードで検証

**TodoItem テスト**
- ✓ 成功メッセージが表示されない
- ✓ 親コンポーネント更新関数が呼ばれない
- ✓ 確認ダイアログが閉じない

**AppEditPage テスト**
- ✓ ローディングメッセージが表示されない
- ✓ エラーアラートが表示される
- ✓ 複数のエラーステータス (404, 500, 403) に対応

---

## 実装時の注意点

### 1. 型キャスト時の nil safety（Null Safety）

応答が `undefined` の可能性を常に想定します：

```typescript
// ❌ 危険
const status = (result as { status: number }).status  // status が undefined なら型エラー

// ✓ 安全
const typedResult = result as { status?: number }
if (typedResult?.status) {  // オプショナルチェーンで nil 安全
  // status が確実に number
}
```

### 2. ステータス範囲の明確化

HTTP ステータスコードの 2xx 範囲は 200-299 です。明示的に範囲チェック：

```typescript
// ✓ 明確で保守性が高い
if (status >= 200 && status < 300) { }

// ❌ 意図が不明確
if (status === 200) { }  // 204 No Content や 201 Created を見落とす
```

### 3. コールバック実行順序

UI 更新のコールバックは状態に応じて実行順序を制御します：

```typescript
// ✓ 正しい順序
if (isSuccess) {
  setSuccessMsg(...)     // 1. UI メッセージ更新
  setShowConfirm(false)  // 2. UI 要素クローズ
  onRefresh()            // 3. 親画面更新
}
```

### 4. エラーメッセージは別途用意

この修正では **ステータス検証のみ** を実装します。エラーメッセージ表示は別の層（グローバル エラー状態管理など）で処理することを想定しています：

```typescript
// 今回の修正：ステータスチェック
if (!isSuccess) {
  return  // 何もしない
}

// 別途実装：エラーメッセージの表示
// useToast().error("保存に失敗しました") など
```

---

## 全体影響度と測定結果

### コード変更の規模

```
修正対象ファイル数: 3
  - TodoForm.tsx
  - TodoItem.tsx
  - AppEditPage.tsx

新規テストケース数: 10
総テストケース数: 128 (変更前 118)

コミット数: 3 (1 修正 = 1 コミット)
```

### 品質指標

| 指標 | 変更前 | 変更後 | 結果 |
|------|-------|--------|------|
| テスト成功 | 118/118 | 128/128 | ✓ 全成功 |
| TypeScript エラー | 0 | 0 | ✓ なし |
| ESLint エラー | 0 | 0 | ✓ なし |

### リスク分析

**低リスク理由：**
- 既存のテストが全 118 件成功（パッドを検証）
- 新テストは既存ロジックを変更しない（追加のみ）
- ステータスチェックロジックは単純（数値比較）
- エラーハンドリングの補強なので機能削減ではない

---

## ビジネスロジック修正から得られた教訓

### 1. API 設計とフロントエンドの契約

API がエラー時に例外をスロー**しない** 場合、フロントエンド側が **必ず** ステータスコードをチェックする必要があります。

**ガイドライン：**
- API 仕様書に「エラーハンドリング方針」を明記
- フロントエンド コンポーネントのテンプレートに「ステータス検証パターン」を組み込み

### 2. ローディング状態とエラー状態の分離

UI 状態管理では、以下を明確に区別する設計を心がけます：

```
isLoading = true   → 処理中
isLoading = false, data = null    → エラー
isLoading = false, data = <obj>   → 成功
```

**チェックリスト：**
- ✓ `isLoading` フラグを常に `true` → `false` へ遷移
- ✓ `error` フィールドがあれば、`!data && !isLoading` より優先
- ✓ ローディング UI と エラー UI を **別の分岐** で描画

### 3. テスト駆動による欠陥検出

これら 3 つの欠陥は、エラーシナリオのテストを追加する過程で検出されました。

**TDD の実践：**
1. 成功シナリオをテスト（実装完了時点）
2. エラーシナリオをテスト（欠陥が明らかになる）
3. 実装を修正（テスト成功）

このサイクルが欠陥を未然に防ぎます。

---

## まとめ

TDD Todo App のフロントエンド 3 つの重要なビジネスロジック修正により：

### 実装した内容

1. **TodoForm**：ミューテーション応答のステータスを検証し、2xx のときのみ成功コールバックを実行
2. **TodoItem**：削除応答のステータスを検証し、2xx のときのみ成功メッセージとリフレッシュを実行
3. **AppEditPage**：`isLoading` を活用してローディング状態とエラー状態を区別

### 達成した効果

- ✓ API エラー (4xx/5xx) が UI に**マスクされない**
- ✓ ユーザーが失敗した操作を**再試行できる**
- ✓ ユーザーが無限ローディング状態に**陥らない**
- ✓ テストカバレッジが 118 → 128 (+10 ケース)
- ✓ 型チェック・リント エラーなし

### 学習ポイント

- API 設計に応じた防御的プログラミング（ステータスチェック）
- UI 状態遷移の明確な設計（isLoading vs. エラー）
- TDD で欠陥を早期検出する仕組み
- React コンポーネントでの安全な型キャスト

---

## 参考資料

- **HTTP ステータスコード**：RFC 7231 Section 6 ([https://tools.ietf.org/html/rfc7231#section-6](https://tools.ietf.org/html/rfc7231#section-6))
- **TDD ガイドライン**：リポジトリ `.github/rules/test-driven-development.rules.md`
- **フロントエンド ルール**：リポジトリ `.github/rules/frontend.rules.md`

---

**作成日**: 2026-04-29  
**対応コミット**: `5844450`, `f159e16`, `6b6450e`
