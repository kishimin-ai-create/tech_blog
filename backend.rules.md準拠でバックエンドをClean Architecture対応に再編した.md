# backend.rules.md に準拠してバックエンドを Clean Architecture 対応に再編した

## 対象読者

- TypeScript + Hono でバックエンドを書いている人
- Clean Architecture を「図」ではなく「ファイル配置と責務」で整えたい人
- `.github/rules/` ベースの設計ルールをコードに落としたい人

## この記事で扱う範囲 / 扱わない範囲

### 扱う範囲

- `.github/rules/backend.rules.md` を基準に、バックエンド実装がどう再編されたか
- 現在の `backend/src/` の層構造と責務分離
- レビュー指摘が今の構造とどうつながっているか
- テスト戦略と CI / package.json への落とし込み

### 扱わない範囲

- MySQL 実接続や ORM 導入
- 認証・認可
- Copilot エージェント運用の詳細

## 先に断っておくこと

本来は `git --no-pager diff --stat` などで差分を確認したいところですが、この記事は `git` コマンドを直接実行できない状況で書かれています。以下の**観測可能なファイル内容**を根拠にしています。

- `backend/src/` の現物コード
- `.github/rules/backend.rules.md`
- `diary/20260425.md`
- `review/todo-api-20260425.md`
- `.github/workflows/backend.yaml`
- `backend/package.json`

つまり、この記事は **git メタデータではなく、現在のリポジトリ状態と関連記録をもとにした技術整理**です。

---

## 背景：「実装が動く」状態から「ルールに沿って保守できる」状態へ

`diary/20260425.md` と過去の TDD 実装記事を読むと、このバックエンドは最初、`backend/src/index.ts` にルーティング・ストレージ・バリデーション・DTO 変換をまとめて持つ形で動いていた。

一方、現在の `backend/src/index.ts` は次のような薄い入口になっている。

```ts
import { createBackendRegistry } from './infrastructure/framework/registry';

const registry = createBackendRegistry();

export function clearStorage(): void {
  registry.clearStorage();
}

export default registry.app;
```

この変化が重要だ。いまの実装は「API が動く」だけでなく、`.github/rules/backend.rules.md` が求める方向へ寄せ直されている。

---

## `backend.rules.md` が何を求めていたか

`backend.rules.md` には、バックエンド実装についてかなり具体的な指針がある。要点を抜くとこうだ。

- `domain` は最内周。フレームワークやインフラに依存しない
- `usecase` は業務フローの調停だけを行う
- `handler/controller` は薄く保つ
- バリデーションは入口側で扱う
- 生のインフラエラーを外へ漏らさない
- registry で依存を接続する
- 検証コマンドは `backend` ディレクトリで `npm run lint` / `npm run typecheck` / `npm run test`

現在の `backend/src/` は、少なくとも観測できる範囲ではこの方針にかなり素直に沿っている。

---

## 現在のバックエンド構成を rules と対応づけて見る

現状の `backend/src/` は次のように分かれている。

```text
backend/src/
├── domain/
│   ├── entities/
│   │   ├── app.ts
│   │   ├── todo.ts
│   │   └── app-error.ts
│   └── repositories/
│       ├── app-repository.ts
│       └── todo-repository.ts
├── usecase/
│   ├── input_ports/
│   │   ├── app-usecase.ts
│   │   └── todo-usecase.ts
│   └── interactors/
│       ├── app-interactor.ts
│       └── todo-interactor.ts
├── interface/
│   ├── controllers/
│   │   ├── app-controller.ts
│   │   ├── todo-controller.ts
│   │   └── request-validation.ts
│   ├── presenters/
│   │   └── http-presenter.ts
│   └── gateways/
│       ├── in-memory-repositories.ts
│       └── in-memory-storage.ts
└── infrastructure/
    └── framework/
        ├── hono-app.ts
        └── registry.ts
```

### 1. `domain/`：エンティティとエラー、Port の定義だけを置く

`app.ts` と `todo.ts` は単純な型定義だ。`deletedAt` を含む内部表現もここで扱っている。

`app-error.ts` には `AppError` があり、コードは次の 4 種類だ。

- `VALIDATION_ERROR`
- `CONFLICT`
- `NOT_FOUND`
- `REPOSITORY_ERROR`

これが後段の層をまたいで流れる「安全な失敗表現」になっている。

`domain/repositories/*.ts` は実装ではなく interface だ。

```ts
export interface AppRepository {
  save(app: AppEntity): Promise<void>;
  listActive(): Promise<AppEntity[]>;
  findActiveById(id: string): Promise<AppEntity | null>;
  existsActiveByName(name: string, excludeId?: string): Promise<boolean>;
}
```

ここには Hono も Map も SQL もない。`backend.rules.md` の「Domain Layer は framework / infrastructure を import しない」に沿った形だ。

### 2. `usecase/`：HTTP を知らない業務フローに閉じ込める

`usecase/input_ports/` では各 usecase の入出力を型として定義している。`interactors/` が実処理だ。

たとえば `app-interactor.ts` の `createApp` では、

- 重複名チェック
- ID 生成
- timestamp 生成
- 保存

といったアプリケーションの流れを担当している。

重要なのは、ここに HTTP レスポンス生成がないことだ。`status code` も `Context` も `c.json()` も出てこない。Usecase は repository port を呼び、必要なら `AppError` を投げるだけだ。

### 3. `interface/controllers/`：本当に薄い controller になった

現在の `app-controller.ts` と `todo-controller.ts` がやっていることはほぼ 3 つだ。

1. request body を parse / validate
2. usecase を呼ぶ
3. `AppError` を HTTP レスポンスへ変換する

```ts
const todo = await todoUsecase.update(
  parseUpdateTodoInput(appId, todoId, body),
);
return presentSuccess(presentTodo(todo));
```

異常系も `handleControllerError()` に寄せられている。「controller が業務ロジックを抱え込まない」形になっている。

### 4. バリデーション境界が `request-validation.ts` に集約された

`backend/src/interface/controllers/request-validation.ts` が、App/Todo の create/update 入力をまとめて扱っている。

- `parseCreateAppInput`
- `parseUpdateAppInput`
- `parseCreateTodoInput`
- `parseUpdateTodoInput`

ここでやっていること：

- body が object でなければ空 object 扱い
- `name` / `title` の trim と長さ制限
- `completed` が boolean 以外なら `VALIDATION_ERROR`

以前レビューで問題になった「`Boolean()` による黙った型変換」を、**controller 手前の validation boundary で止める**方向に変わっている。

### 5. `interface/presenters/`：DTO と HTTP エラー写像をここに寄せる

`http-presenter.ts` では `presentApp`、`presentTodo`、`presentSuccess`、`presentError` が定義されている。

#### `deletedAt` をレスポンスに出さない

内部 entity には `deletedAt` があるが、外向け DTO には出していない。これは `docs/design/backend/api.md` の「GET responses do NOT include `deletedAt`」という仕様と一致する。

#### エラーコードから HTTP status への変換を presenter に閉じ込める

```
VALIDATION_ERROR → 422
CONFLICT → 409
NOT_FOUND → 404
REPOSITORY_ERROR → 500
```

usecase 側は HTTP を知らずに済み、controller 側も `presentError()` を呼ぶだけで済む。

### 6. `interface/gateways/`：repository 実装は Port の裏側に押し込める

`in-memory-repositories.ts` では `Map` を直接触りながらも、例外は `withRepositoryError()` で包んでいる。

```ts
function withRepositoryError<T>(operation: () => T): T {
  try {
    return operation();
  } catch {
    throw new AppError('REPOSITORY_ERROR', 'Repository operation failed');
  }
}
```

生の例外をそのまま外に漏らさず、`backend.rules.md` の「Never leak raw infrastructure errors outside the infrastructure layer.」に近い運用ができている。

現時点では DB ではなく `Map` ベースだが、**Port の背後に隠してある**ので、将来 MySQL 実装へ差し替える場所も明確だ。

### 7. `infrastructure/framework/`：Hono と DI 配線だけを置く

`hono-app.ts` はルート登録専用だ。Hono の `Context` を触るのはここだけで、各ルートは controller に委譲している。

`registry.ts` が依存を配線している。

- storage 作成 → repository 作成 → interactor 作成 → controller 作成 → Hono app 作成

この registry があることで `index.ts` は薄く保てている。`backend.rules.md` にある「Registry: Wire everything together using Dependency Injection」に対応する実装だ。

---

## 旧実装と比べると、何がいちばん変わったのか

いちばん大きいのは、**責務の置き場所が明確になったこと**だ。

過去の実装では `backend/src/index.ts` が次をまとめて持っていた。

- ルーティング
- ストレージ
- バリデーション
- DTO 変換
- エラーレスポンス生成
- カスケード削除

現在はこれが分散されている。

| 責務 | 現在の主な配置 |
|---|---|
| Entity / エラー | `domain/entities/` |
| Repository Port | `domain/repositories/` |
| 業務フロー | `usecase/interactors/` |
| 入力境界 | `interface/controllers/request-validation.ts` |
| HTTP controller | `interface/controllers/*.ts` |
| DTO / レスポンス整形 | `interface/presenters/http-presenter.ts` |
| repository 実装 | `interface/gateways/*.ts` |
| Hono と配線 | `infrastructure/framework/*.ts` |

この分解により、少なくとも次の利点がある。

- validation の修正箇所が明確
- HTTP 仕様変更が usecase に波及しにくい
- DB 置換時に controller/usecase まで巻き込みにくい
- review で「どの層の責務違反か」を判断しやすい

---

## レビューで出た指摘が、今の構造とどうつながっているか

`review/todo-api-20260425.md` は、単にバグ一覧ではなく「どこで受けるべき問題か」を示している。

### 1. `completed` の `Boolean()` 強制変換問題

レビューでは、`"false"` や `1` を `Boolean()` が黙って変換してしまう問題が指摘されていた。

現在の `request-validation.ts` では次の形で止められている。

```ts
if (payload.completed !== undefined) {
  if (typeof payload.completed !== 'boolean') {
    throw new AppError('VALIDATION_ERROR', 'completed must be a boolean');
  }
  input.completed = payload.completed;
}
```

これは単なるバグ修正ではなく、**validation を interface 層で受ける**という rules 準拠そのものだ。

### 2. trim と一意制約の不一致

レビューでは、trim して空文字判定はしているのに、保存や重複判定は元の文字列で行っていた点も指摘されていた。

現在の validation は `trimmed` を返すので、空白だけを reject・長さ判定も trim 後・保存される値も trim 済みという流れになっている。

### 3. `createdAt` / `updatedAt` の timestamp 二重取得

`now()` を 2 回呼ぶのではなく 1 回にまとめるべきという指摘も入っていた。現在の `app-interactor.ts` / `todo-interactor.ts` は `const timestamp = now();` と 1 回だけ取得している。

### 4. まだ残るテスト上の弱さ

cascade delete テストの弱さは、現状でも完全には消えていない。`backend/src/app.test.ts` の該当テストには、「The todo belonged to the old appId; either 404 app or 404 todo is acceptable」というコメントが入っており、直接的に cascade の内部状態を証明しているわけではないことが自覚的に残されている。

---

## テスト戦略も、層分離後の形に合っている

### `backend/src/index.test.ts`：入口のスモークテスト

```ts
describe('GET /', () => {
  it('when the root endpoint is requested, then it returns the greeting text', async () => {
    const response = await app.request('http://localhost/');
    expect(response.status).toBe(200);
    await expect(response.text()).resolves.toBe('Hello Hono!');
  });
});
```

Hono アプリが正しく立ち上がっているかの最低限の確認だ。

### `backend/src/app.test.ts`：HTTP レベルの統合テスト

App/Todo の CRUD を HTTP 経由で広く押さえている。

観測できるケースだけでも：App 作成・一覧・単体取得・更新・削除、Todo 作成・一覧・単体取得・更新・削除、422 系入力エラー、409 重複、404 存在しない ID、soft delete 後の非表示、DTO に `deletedAt` を出さないこと、100 文字 / 200 文字境界値、`updatedAt >= createdAt` まで見ている。

### `clearStorage()` を registry 越しに公開しているのが地味に良い

`beforeEach(() => clearStorage())` のために `index.ts` が test helper を export している。しかも storage を直接 export せず `registry.clearStorage()` に閉じているので、テスト都合の API 追加が内部構造を壊しにくい。

---

## `backend.rules.md` の検証コマンドは package.json と CI にまで落ちている

`backend.rules.md` には、検証時は `backend/` で次を実行するよう書かれている。

- `npm run lint`
- `npm run typecheck`
- `npm run test`

現在の `backend/package.json` はそのままこれを持っている。

```json
"scripts": {
  "dev": "bun run --hot src/index.ts",
  "lint": "eslint .",
  "test": "vitest run --reporter=default --reporter=json --outputFile=test-result.json",
  "typecheck": "tsc --noEmit"
}
```

この `test` が JSON レポートも出すようになっているのが CI につながる。

`.github/workflows/backend.yaml` では：

- `backend` で `npm ci`
- `npm run typecheck`
- `npm run lint`
- `npm test`
- `backend/test-result.json` を読んで PR にコメント

という流れになっている。つまり「テストを走らせる」だけでなく、**PR 上で見える形にまで戻す**ところまで workflow 化されている。

---

## TypeScript ルール追加も、今回のバックエンド再編と相性がいい

`.github/rules/typescript.rules.md` では、

- `as` 禁止
- `any` 禁止
- やむを得ず使うなら直前コメント必須
- 代替として `unknown` と型ガードを推奨

が定義されている。

今回の backend 側で見ると、`request-validation.ts` は `unknown` を受けて `typeof`・`Array.isArray`・`object` 判定で絞り込みながら parse している。Clean Architecture の話に見えて、実際にはこうした TypeScript ルールも「境界で安全に扱う」設計と噛み合っている。

---

## 今の構成で見えている限界

### 1. `usecase/output_ports/` は現状見当たらない

`backend.rules.md` の例では `usecase/output_ports/` があるが、現在の `backend/src/usecase/` にあるのは `input_ports/` と `interactors/` だけだ。現状は presenter 側でレスポンス DTO を組み立てているため成立しているが、**ルール例を完全にそのまま写した構成ではない**。

### 2. DB 実装はまだ in-memory

`interface/gateways/` の実装は `Map` ベースだ。`docs/design/db/db.md` には MySQL スキーマがあるので、将来的には DB gateway / infrastructure database 層が増えるはずだが、そこはまだ未実装だ。

### 3. 一部テストはさらに強くできる

- cascade delete の直接証明
- `completed` 非 boolean の 422
- repository error の 500 応答

あたりは今後の追加候補として見える。

### 4. 実コマンド実行による確認はしていない

`backend/package.json` と workflow 定義から `lint` / `typecheck` / `test` の導線は確認できるが、この記事作成時点ではそれらのコマンド実行結果までは確認していない。ここで言えるのは「実行経路が定義されている」までだ。

---

## まとめ

今回の変化をひとことで言うなら、**Todo API を作った**だけではなく、**その実装を repository ルールに従って保守可能な形へ載せ替えた**ことが本質だ。

- `backend/src/index.ts` に集まっていた責務を `domain/` `usecase/` `interface/` `infrastructure/` に分離した
- controller は「validate → usecase 呼び出し → presenter 返却」の薄い形に寄せた
- repository 実装の例外は `AppError('REPOSITORY_ERROR')` に写像し、生の失敗を漏らしにくくした
- response shaping と error-to-status mapping を presenter に寄せた
- registry で依存を配線し、入口を薄くした
- テスト、レビュー、CI、PR コメントまでをひとつの流れにした

個人的にいちばん良いと思ったのは、`.github/rules/backend.rules.md` が単なる理想論ではなく、現在のファイル配置、エラーハンドリング、workflow にまで落ちている点だ。

設計ルールは、書くだけでは効かない。**どの層に何を書くか、失敗をどこで止めるか**までつながって、はじめて repository の習慣になる。
