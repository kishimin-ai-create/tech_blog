# Hono + Kysely API でのアプリと Todo のユーザー所有権の強制

## 対象読者

- Hono を使用してマルチユーザー REST API を構築するバックエンド エンジニア
- ベアラートークン認証とリソースごとの認可を既存のコードベースに追加するためのクリーンなパターンを探しているエンジニア
- TypeScript に精通しており、所有権チェックが階層化された (インタラクター / リポジトリ / コントローラー) アーキテクチャにどのように適合するかを知りたい開発者

---

## 問題の背景

Todo API は、クライアントが ID チェックなしでアプリや Todo を作成、読み取り、更新、削除できるシングルユーザー アプリケーションとして始まりました。このモデルは、2 人目のユーザーがサインアップした瞬間に崩れます。すべてのアプリが全員に表示され、どのユーザーも別のユーザーのデータを変更または削除できるようになります。

解決する必要がある 2 つの具体的な問題:

1. **認証** — 保護されたエンドポイントへのすべてのリクエストには有効なトークンが含まれている必要があり、サーバーはそのトークンを実際のユーザーに解決する必要があります。
2. **認可** — 認証に合格した後でも、サービス層は操作を許可する前に、認証されたユーザーが実際にアプリ (または Todo を含むアプリ) を所有していることを確認する必要があります。

---

## ソリューションの概要

この修正は、バックエンド スタックのすべての層 (データベース、リポジトリ、サービス、コントローラー、HTTP ルーター) に加え、トークンを自動的にアタッチするようにフロントエンドに小さな変更を加えました。

|レイヤー |変更 |
|---|---|
|データベース | `userId` 列が `App` テーブルに追加されました。
|リポジトリ | `listActive` → `listActiveByUserId`; `existsActiveByName` のスコープはユーザーです。 `UserRepository.findByToken` が追加されました |
|サービス | `findAccessibleApp` (所有権チェック) `AppInteractor`; `TodoInteractor`の`ensureAppBelongsToUser` |
|エラーモデル | `FORBIDDEN` エラー コードが追加されました |
| HTTPルーター | `authMiddleware` がすべての保護されたルートに追加されます。 `userId` コンテキストから伝播 |
|フロントエンド | `customFetch` は、`localStorage` からベアラー トークンを読み取り、それを `Authorization` ヘッダーとして挿入します。

---

## 実装の詳細

### 1. データベースの移行

```sql
-- backend/migrations/004_add_userId_to_app.sql
ALTER TABLE App
  ADD COLUMN userId CHAR(36) NOT NULL DEFAULT '' AFTER id,
  ADD INDEX idx_app_userId (userId);
```

この列は、非 NULL 制約付きで `id` の直後に配置されます。 `userId` のインデックスにより、ユーザーごとのアプリのリスト クエリが効率的に行われます。

### 2.AppEntityModel

```ts
// backend/src/models/app.ts
export type AppEntity = {
  id: string;
  userId: string;      // <-- added
  name: string;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null;
};
```

### 3. リポジトリインターフェースの更新

```ts
// backend/src/repositories/app-repository.ts
export interface AppRepository {
  save(app: AppEntity): Promise<void>;
  listActiveByUserId(userId: string): Promise<AppEntity[]>; // renamed + scoped
  findActiveById(id: string): Promise<AppEntity | null>;
  existsActiveByName(name: string, userId: string, excludeId?: string): Promise<boolean>; // userId added
}
```

所有者に関係なくすべてのアプリを誤って取得できないようにするために、`listActive()` は `listActiveByUserId()` になりました。 `existsActiveByName` は `userId` も受け取るので、2 人の異なるユーザーが競合を引き起こすことなく同じ名前のアプリを作成できます。

```ts
// backend/src/repositories/user-repository.ts
export interface UserRepository {
  findByEmail(email: string): Promise<UserEntity | null>;
  findByToken(token: string): Promise<UserEntity | null>; // <-- added
  save(user: UserEntity): Promise<void>;
}
```

`findByToken` は、認証ミドルウェアが生のベアラー トークンを `UserEntity` に解決するために使用するルックアップです。

### 4. インメモリリポジトリの実装

メモリ内実装 (テストで使用) はコントラクトを反映しています。

```ts
// listActiveByUserId filters by userId and non-deleted
function listActiveByUserId(userId: string): Promise<AppEntity[]> {
  return Promise.resolve().then(() =>
    withRepositoryError(() =>
      [...storage.apps.values()]
        .filter(app => app.userId === userId && app.deletedAt === null)
        .map(cloneApp),
    ),
  );
}

// existsActiveByName now scopes the duplicate check to the same user
function existsActiveByName(
  name: string,
  userId: string,
  excludeId?: string,
): Promise<boolean> {
  return Promise.resolve().then(() =>
    withRepositoryError(() =>
      [...storage.apps.values()].some(
        app =>
          app.userId === userId &&
          app.deletedAt === null &&
          app.name === name &&
          app.id !== excludeId,
      ),
    ),
  );
}

// findByToken scans the user store for a matching token
function findByToken(token: string): Promise<UserEntity | null> {
  return Promise.resolve().then(() =>
    withRepositoryError(() => {
      const user = [...store.values()].find(u => u.token === token);
      return user ? { ...user } : null;
    }),
  );
}
```

### 5. 禁止されたエラーコード

```ts
// backend/src/models/app-error.ts
export type AppErrorCode =
  | 'CONFLICT'
  | 'NOT_FOUND'
  | 'UNAUTHORIZED'
  | 'FORBIDDEN'       // <-- added
  | 'REPOSITORY_ERROR';
```

`UNAUTHORIZED` は、「あなたが誰であるかを証明しませんでした」(トークンが存在しないか無効である) を意味します。 `FORBIDDEN` は、「私たちはあなたが誰であるかを知っていますが、あなたはこのリソースを所有していません」を意味します。それらを分離しておくと、HTTP レイヤーが正しいステータス コード (`401` と `403`) を返すことができます。

### 6. サービス層での所有権チェック

**AppInteractor** — 単一のプライベート ヘルパー `findAccessibleApp` が古い `findExistingApp` を置き換えます。 `NOT_FOUND` ガードの後に​​追加のチェックが 1 つ追加されます。

```ts
async function findAccessibleApp(input: {
  appId: string;
  userId: string;
}): Promise<AppEntity> {
  const app = await appRepository.findActiveById(input.appId);
  if (!app) throw new AppError('NOT_FOUND', 'App not found');
  if (app.userId !== input.userId) {
    throw new AppError('FORBIDDEN', 'Access denied');
  }
  return app;
}
```

すべての変更操作 (`get`、`update`、`delete`) は、`findExistingApp` ではなく `findAccessibleApp` を呼び出すようになりました。 `create` 操作は、新しいエンティティに `userId` を設定し、それを `existsActiveByName` を通じて渡して重複検出の範囲を設定します。

**TodoInteractor** — Todo は `userId` を直接所有しません。それらはアプリに属しています。したがって、ガード `ensureAppBelongsToUser` は、所有権を強制するのに適切な場所です。

```ts
async function ensureAppBelongsToUser(
  appId: string,
  userId: string,
): Promise<AppEntity> {
  const app = await appRepository.findActiveById(appId);
  if (!app) throw new AppError('NOT_FOUND', 'App not found');
  if (app.userId !== userId) throw new AppError('FORBIDDEN', 'Access denied');
  return app;
}
```

5 つの Todo 操作 (`create`、`list`、`get`、`update`、`delete`) はすべて、続行する前にこのガードを呼び出します。

### 7. Honoのベアラートークンミドルウェア

認証ミドルウェアは保護されたルートごとに登録されます。 `Authorization` ヘッダーを読み取り、`Bearer ` プレフィックスを取り除き、`userRepository.findByToken` を介してトークンをユーザーに解決します。成功すると、ユーザー ID が Hono コンテキスト変数 `userId` に保存され、コントローラーがそれを読み取れるようになります。

```ts
// backend/src/infrastructure/hono-app.ts (excerpt)

type AppEnv = {
  Variables: {
    userId: string;
  };
};

async function authMiddleware(
  c: Context<AppEnv>,
  next: Next,
): Promise<Response | void> {
  const authHeader = c.req.header('Authorization');
  if (!authHeader?.startsWith('Bearer ')) {
    return c.json(
      { success: false, data: null, error: { code: 'UNAUTHORIZED', message: 'Authentication required' } },
      401,
    );
  }

  const token = authHeader.slice(7);
  const user = await userRepository.findByToken(token);
  if (!user) {
    return c.json(
      { success: false, data: null, error: { code: 'UNAUTHORIZED', message: 'Invalid or expired token' } },
      401,
    );
  }

  c.set('userId', user.id);
  await next();
}
```

Hono ジェネリック `Hono<AppEnv>` はコンテキスト変数の型をアプリに結び付けるため、`c.get('userId')` はキャストなしのすべてのルート ハンドラーにわたって完全に型安全です。

CORS `allowHeaders` も更新され、`Authorization` が含まれました。

```ts
allowHeaders: ['Content-Type', 'Authorization'],
```

これがないと、フロントエンドが認証ヘッダーを送信するときに、ブラウザーはプリフライト要求を拒否します。

### 8. コントローラーとルートの配管

コントローラーは `userId` をパラメーターとして受け入れ、それをユースケース入力に転送するようになりました。

```ts
// app-controller create example (conceptual)
async create(body: unknown, userId: string): Promise<PresenterResult> { ... }
```

ルート ハンドラーは Hono コンテキストから `userId` を抽出し、それを渡します。

```ts
app.post('/apps', authMiddleware, async c =>
  toJsonResponse(c, await appController.create(await readRequestBody(c), c.get('userId'))),
);
```

### 9. フロントエンド — 自動トークンインジェクション

集中型の `customFetch` ラッパーは、キー `auth` の下の `localStorage` に以前に保存されていた認証状態を読み取り、`token` を抽出し、それをすべての送信リクエストに挿入します。

```ts
// frontend/src/api/client.ts (excerpt)
const authRaw = localStorage.getItem('auth');
let auth: AuthState | null = null;
if (authRaw) {
  try { auth = JSON.parse(authRaw) as AuthState | null; } catch { auth = null; }
}

const headers: Record<string, string> = {
  ...(options?.headers as Record<string, string> | undefined),
};
if (auth?.token) {
  headers.Authorization = `Bearer ${auth.token}`;
}

const response = await fetch(`${BASE_URL}${url}`, { ...options, headers });
```

すべての API 呼び出しは `customFetch` を経由するため、この 1 つの変更で既存のすべての呼び出しを認証するのに十分です。個々のクエリ/ミューテーション フックに変更を加える必要はありません。

---

## テスト

2 つの新しいテスト シナリオが追加されました。

- **未認証リクエスト → 401** - `Authorization` ヘッダーなしでリクエストを送信すると、`{ code: "UNAUTHORIZED" }` とステータス `401` を返す必要があります。
- **クロスユーザー リソース アクセス → 403** - ユーザー B が所有するリソースをターゲットとしてユーザー A に有効なトークンを送信すると、`{ code: "FORBIDDEN" }` とステータス `403` が返される必要があります。

既存のテストは、新しい入力タイプで必要な場合に `userId` を提供するように更新されました。テスト スイート全体で使用されるメモリ内リポジトリは、MySQL 実装と同じように動作しますが、データベースへの依存はありません。

---

## 注意すべき点

### アプリ名の一意性はユーザーごとになりました

この変更が行われる前は、2 つのアプリがシステム全体で同じ名前を共有することはできませんでした。変更後は、一意性の範囲は所有ユーザーに限定されます。これはほぼ確実に製品の正しい動作ですが、既存のデータや統合テストでは予期しないセマンティックな変更である可能性があります。

### `findByToken` はインメモリ実装でフル スキャンを実行します

インメモリ `findByToken` は、すべてのユーザーを反復してトークンの一致を見つけます。これはテストでは許容されますが、トークン フィールドにまだインデックスが作成されていない場合、MySQL 実装ではインデックス付きカラムを使用する必要があります。

### 移行時のデフォルトの `userId`

移行により、既存の行に `DEFAULT ''` が設定されます。既存のアプリ レコードには、`userId` として空の文字列が含まれます。実稼働データがどのようにシードされたかによっては、実際のユーザー ID をバックフィルするためにフォローアップのデータ移行が必要になる場合があります。

### CORS には `Authorization` を含める必要があります

`allowHeaders` の `Authorization` を忘れると、トークンがチェックされる前に、すべてのブラウザーのプリフライトが CORS エラーで失敗します。この修正はこのコミットに含まれていますが、既存の API に認証を追加するときに見逃されがちです。

---

## まとめ

このコミットは、データベース スキーマから、リポジトリ インターフェイス、サービス層の所有権チェックを経て、Hono ルート ハンドラーに至るまで、あらゆる層に `userId` をスレッド化することで、Todo API の承認ギャップを埋めます。設計上の主な決定事項は次のとおりです。

- **HTTP 層で認証を維持** (`authMiddleware`) し、`userId` をプレーン値としてサービス層に渡し、ビジネス ロジックを HTTP の問題から解放します。
- **コントローラーでチェックを重複させるのではなく、ドメイン ルールが存在するサービス レイヤー** (`findAccessibleApp`、`ensureAppBelongsToUser`) で承認を維持します。
- **クライアントとオペレーターに意味のある診断信号を提供するために、`UNAUTHORIZED` を `FORBIDDEN` から分離します**。
- **マルチテナントの自然な結果として、重複名の検出の範囲をユーザーに限定します**。
- **フロントエンドの `customFetch`** にトークン インジェクションを一元化することで、個々のフックを変更する必要がなくなります。
