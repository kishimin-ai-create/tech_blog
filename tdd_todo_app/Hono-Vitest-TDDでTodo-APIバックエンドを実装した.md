# Hono + Vitest で TDD：Todo API バックエンドを Red→Green→Refactor→Review の4フェーズで実装した

## 対象読者

- TypeScript + Hono でバックエンドを書いている、または興味がある人
- テスト駆動開発（TDD）を実務で試してみたい人
- REST API のテスト設計に迷っている人

## この記事で扱うこと / 扱わないこと

**扱う**
- TDD の4フェーズ（Red / Green / Refactor / Review）を通した実装の流れ
- Hono アプリに対する Vitest 統合テストの書き方
- インメモリストアを使ったテスト可能な設計
- コードレビューで見つかった2種類の実際のバグとその修正

**扱わない**
- 実際のデータベース（MySQL など）との接続
- 認証・認可
- OpenAPI/Swagger の生成

---

## 背景

今回実装したのは、複数の「App（ワークスペース）」とそこに紐づく「Todo」を管理する REST API だ。設計仕様はすでに `docs/design/backend/api.md` として存在しており、それを TDD サイクルで実装することが目標だった。

スタックは次のとおり。

| 役割 | ライブラリ |
|---|---|
| Web フレームワーク | [Hono](https://hono.dev/) |
| テストランナー | [Vitest](https://vitest.dev/) |
| ランタイム | Bun |
| バリデーション | 自前（Zod は今回不使用） |

---

## API 仕様の概要

ベース URL は `/api/v1`。エンドポイントは10本。

```
# App
POST   /api/v1/apps
GET    /api/v1/apps
GET    /api/v1/apps/:appId
PUT    /api/v1/apps/:appId
DELETE /api/v1/apps/:appId        ← ソフトデリート。配下の Todo をカスケード削除

# Todo（App に紐づく）
POST   /api/v1/apps/:appId/todos
GET    /api/v1/apps/:appId/todos
GET    /api/v1/apps/:appId/todos/:todoId
PUT    /api/v1/apps/:appId/todos/:todoId
DELETE /api/v1/apps/:appId/todos/:todoId  ← ソフトデリート
```

レスポンスは常に次の2形式のどちらかで統一する。

```json
// 成功
{ "data": { ... }, "success": true }

// エラー
{ "data": null, "success": false, "error": { "code": "ERROR_CODE", "message": "..." } }
```

`deletedAt` フィールドはレスポンスに含めない（ソフトデリートはDBレイヤーの内部実装として扱う）。

---

## 🔴 Red フェーズ：先にテストを書く

### テストファイルの骨格

`backend/src/app.test.ts` を新規作成した。Hono はサーバーを立てずに `app.request()` でリクエストを送れるため、純粋なユニットテストとほぼ同じ感覚でエンドツーエンドの HTTP テストが書ける。

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import app, { clearStorage } from './index';

const request = (method: string, path: string, body?: unknown) =>
  app.request(`http://localhost${path}`, {
    method,
    headers: body !== undefined ? { 'Content-Type': 'application/json' } : undefined,
    body: body !== undefined ? JSON.stringify(body) : undefined,
  });

beforeEach(() => clearStorage());
```

`clearStorage` は実装側からエクスポートするテスト用ヘルパーだ。`beforeEach` で呼ぶことで、各テストが完全に独立した状態から始まる。これがないと「重複チェック」や「一覧が空であること」を確認するテストが他のテストの実行結果に依存してしまう。

### テストの記述パターン

`it` のラベルは `"when [条件], then [期待結果]"` の形を基本にした。

```typescript
describe('POST /api/v1/apps', () => {
  it('201: creates an app with correct response shape', async () => {
    const res = await request('POST', '/api/v1/apps', { name: 'My App' });

    expect(res.status).toBe(201);
    const json = await res.json();
    expect(json.success).toBe(true);
    expect(UUID_RE.test(json.data.id)).toBe(true);
    expect(json.data.name).toBe('My App');
    expect(ISO8601_RE.test(json.data.createdAt)).toBe(true);
    expect(json.data.deletedAt).toBeUndefined(); // レスポンスに含まれないこと
  });

  it('422: whitespace-only name', async () => {
    const res = await request('POST', '/api/v1/apps', { name: '   ' });
    expect(res.status).toBe(422);
  });

  it('409: duplicate app name', async () => {
    await request('POST', '/api/v1/apps', { name: 'Duplicate App' });
    const res = await request('POST', '/api/v1/apps', { name: 'Duplicate App' });
    expect(res.status).toBe(409);
  });
});
```

**この時点では実装がないため、すべてのテストが `404 Not Found` で失敗する。** これが Red フェーズの正しい状態だ。

### カバレッジの設計方針

| テスト種別 | 例 |
|---|---|
| Happy path | 正常な入力 → 正しいステータスコードとレスポンス形状 |
| 境界値 | 名前がちょうど100文字 → 201 / 101文字 → 422 |
| バリデーションエラー | 欠如・空文字・空白のみ → 422 |
| 一意制約違反 | 重複名 → 409 |
| 404 | 存在しない ID → 404 |
| ソフトデリート後 | 削除済みリソースが一覧から消える・単体取得が404になる |
| カスケードデリート | App 削除後に配下の Todo も参照不可になる |

---

## 🟢 Green フェーズ：テストを通す実装

### インメモリストア

本番では MySQL を使うが、テストフェーズでは DB 接続は不要だ。TypeScript の `Map` でシンプルに代替する。

```typescript
type AppRecord = {
  id: string; name: string
  createdAt: string; updatedAt: string; deletedAt: string | null
}

type TodoRecord = {
  id: string; appId: string; title: string; completed: boolean
  createdAt: string; updatedAt: string; deletedAt: string | null
}

const appsStore  = new Map<string, AppRecord>()
const todosStore = new Map<string, TodoRecord>()

export const clearStorage = () => { appsStore.clear(); todosStore.clear() }
```

`clearStorage` を named export することで、テストの `beforeEach` から呼べるようにした。

### レスポンスエンベロープとDTO

内部の `Record` 型（`deletedAt` を持つ）とレスポンス（持たない）を分離するため、DTO変換関数を用意した。

```typescript
const toAppDto  = (a: AppRecord)  => ({ id: a.id, name: a.name, createdAt: a.createdAt, updatedAt: a.updatedAt })
const toTodoDto = (t: TodoRecord) => ({ id: t.id, appId: t.appId, title: t.title, completed: t.completed, createdAt: t.createdAt, updatedAt: t.updatedAt })
```

### バリデーション

```typescript
const validateName = (name: unknown): string | null => {
  if (typeof name !== 'string') return 'name is required'
  const trimmed = name.trim()
  if (trimmed.length === 0) return 'name must not be empty'
  if (trimmed.length > 100) return 'name must be at most 100 characters'
  return null
}
```

エラーがあればメッセージを文字列で返し、問題なければ `null` を返す設計にした。呼び出し側は `if (err) return 422` と一行で済む。

### ソフトデリートとカスケード

DELETE エンドポイントでは `deletedAt` にタイムスタンプをセットするだけだ。カスケード削除も同じループで処理できる。

```typescript
app.delete('/api/v1/apps/:appId', (c) => {
  const a = findApp(c.req.param('appId'))
  if (!a) return c.json(errorRes('NOT_FOUND', 'App not found'), 404)

  const ts = now()
  a.deletedAt = ts
  for (const t of todosStore.values()) {
    if (t.appId === a.id && t.deletedAt === null) t.deletedAt = ts
  }
  return c.json(successRes(toAppDto(a)))
})
```

---

## 🔵 Refactor フェーズ

実装を見直したが、ヘルパー関数の切り出しやDTOの分離、ルートのセクション分けが既にできており、追加のリファクタリングは不要だった。

---

## 🔍 Review フェーズ：静的解析で見つかった2つのバグ

コードレビュー（`review/todo-api-20260425.md`）で、自動実装後に2件の実際のバグが検出された。

---

### バグ1：`Boolean()` 強制変換による型安全の破綻（P2）

**問題のコード（修正前）**

```typescript
if (body.completed !== undefined) {
  t.completed = Boolean(body.completed)  // ← 危険
}
```

`Boolean()` はあらゆる truthy/falsy 値をそのまま変換してしまう。

| クライアントが送った値 | `Boolean()` の結果 | 期待される動作 |
|---|---|---|
| `"false"` （文字列）| `true` ✗ | 422 を返すべき |
| `1` （数値）| `true` ✗ | 422 を返すべき |
| `null` | `false` ✗ | 422 を返すべき |

`{ "completed": "false" }` というよくあるミスが、エラーを返さず Todo を「完了済み」にしてしまう**サイレントなデータ破壊バグ**だ。

**修正後**

```typescript
if (body.completed !== undefined) {
  if (typeof body.completed !== 'boolean') {
    return c.json(errorRes('VALIDATION_ERROR', 'completed must be a boolean'), 422)
  }
  t.completed = body.completed  // この時点で型は確実に boolean
}
```

`typeof` で厳密チェックしてから代入する。`Boolean()` は便利に見えるが、外部入力のバリデーションには使ってはいけない。

---

### バグ2：ホワイトスペースが保存前にトリムされていない（P3）

**問題のコード（修正前）**

```typescript
const validateName = (name: unknown): string | null => {
  if (name.trim().length === 0) return 'name must not be empty'  // チェックはトリム後
  if (name.length > 100) ...                                     // 長さはトリム前 ← 不整合
  return null
}
// そして保存時も
const name = body.name as string  // トリム前の値をそのまま保存
```

チェックはトリム後の文字列で行うのに、**保存はトリム前の元の文字列**で行っていた。これにより：

1. `"  My App  "` が通過して前後にスペース付きで保存される
2. 後から `"My App"` で POST すると一意制約が通ってしまう（同名2件が存在できる）

**修正後**

```typescript
const validateName = (name: unknown): string | null => {
  if (typeof name !== 'string') return 'name is required'
  const trimmed = name.trim()
  if (trimmed.length === 0) return 'name must not be empty'
  if (trimmed.length > 100) return 'name must be at most 100 characters'
  return null
}
// 保存時
const name = (body.name as string).trim()  // トリム済みを保存
```

バリデーション境界でトリムし、その値をそのまま保存する。`validateTitle` も同様に修正した。

---

## まとめ

### TDD サイクルを通して得たポイント

1. **テストファースト**は「仕様を実行可能なコードとして書く」行為だ。  
   `POST /api/v1/apps` に対して「409 が返ること」をテストで書くと、仕様の意図が曖昧な部分が自然と浮かび上がる。

2. **`clearStorage` パターン**はインメモリテストの定番。  
   `beforeEach` でリセットすることで、テストの実行順序に依存しない独立性が担保される。

3. **Hono の `app.request()`** はサーバーレスでテストできる理想的な設計。  
   ポートを開かずに HTTP レベルのアサーションが書けるので、CI でも高速に動く。

4. **`Boolean()` は外部入力バリデーションに使わない。**  
   `typeof x === 'boolean'` か Zod の `z.boolean()` を使う。

5. **validate して保存するまで `trim()` を一貫させる。**  
   チェックとストレージで使う文字列が異なると、一意制約が形骸化する。

### 生成されたファイル

```
backend/src/app.test.ts         ← 新規（テストスイート）
backend/src/index.ts            ← 新規実装（全10ルート + レビュー修正適用済み）
review/todo-api-20260425.md     ← コードレビュー結果
```
