# Fix: Render フリープランで本番起動が DB 設定エラーで即クラッシュしていた問題

**Date:** 2026-05-11
**Tech stack:** Node.js · TypeScript · Hono · MySQL2 · Vitest · Render.com
**Changed files:**
- `backend/src/index.ts`
- `backend/src/infrastructure/migrate.ts`
- `backend/src/index.small.test.ts`（新規追加）

---

## Error Overview

Render のフリープランにデプロイした直後、バックエンドが起動した瞬間に以下のエラーで即死していた。

```
Error: DB_USERNAME and DB_PASSWORD environment variables are required
       when DATABASE_URL is not set
    at getMysqlConnectionConfig (…/backend/dist/server.mjs:1073:11)
    at createMysqlPool (…/backend/dist/server.mjs:1088:18)
    at createMysqlBackendRegistry (…/backend/dist/server.mjs:1103:16)
```

加えて、`render.yaml` に記述してあったビルドステップの `node dist/infrastructure/migrate.mjs` も同じ `getMysqlConnectionConfig()` を呼び出すため、**ビルドフェーズでもクラッシュ**していた。

---

## Cause

### MySQL レジストリを無条件で呼び出していた

`backend/src/index.ts` は起動時に次のような分岐を持っていた。

```typescript
// ❌ 修正前
const isTest = process.env.NODE_ENV === 'test';

if (isTest) {
  const registry = createBackendRegistry();       // インメモリ
  honoApp = registry.app;
  clearStorageFn = registry.clearStorage;
} else {
  const registry = createMysqlBackendRegistry();  // MySQL — DB 設定がないとここで throw
  honoApp = registry.app;
}
```

`NODE_ENV === 'test'` でなければ、**常に** `createMysqlBackendRegistry()` が呼ばれる。この関数の内部では `getMysqlConnectionConfig()` が `DATABASE_URL` か `DB_USERNAME` の存在を必須チェックしており、どちらもセットされていない場合は例外を投げる。

### Render フリープランの制約

Render のフリープランが提供するデータベースは **PostgreSQL のみ**で、MySQL は含まれない。
また、Render ダッシュボードで MySQL 用の環境変数（`DATABASE_URL` / `DB_USERNAME` / `DB_PASSWORD`）を設定していなかったため、本番環境では DB 設定が一切存在しない状態だった。

```
NODE_ENV=production
DATABASE_URL  → (未設定)
DB_USERNAME   → (未設定)
```

この組み合わせにより、`createMysqlBackendRegistry()` は呼ばれた瞬間に例外を投げ、プロセスがクラッシュした。

### migrate.ts も同じ原因でクラッシュ

`render.yaml` のビルドコマンドは次の形になっていた。

```yaml
buildCommand: npm run build && node dist/infrastructure/migrate.mjs
```

`migrate.mjs` は `getMysqlConnectionConfig()` を直接呼び出していたため、
DB 設定がない Render 環境ではビルドフェーズでも同様にクラッシュした。

---

## Fix

### `backend/src/index.ts` — `resolveApp()` の導入と DB 設定チェック

ロジックを `resolveApp()` 関数に切り出し、MySQL を使う条件を **「テストでない」かつ「DB 設定が存在する」** に絞った。

```typescript
// ✅ 修正後
type ResolvedApp = {
  app: Hono;
  clearStorage?: () => void;
};

/**
 * 現在の環境に応じて適切なバックエンドレジストリを解決する。
 * DB 設定がある場合は MySQL を使い、なければインメモリにフォールバックする。
 */
export function resolveApp(): ResolvedApp {
  const isTest = process.env.NODE_ENV === 'test';
  const hasDatabaseConfig = !!(process.env.DATABASE_URL || process.env.DB_USERNAME);

  if (!isTest && hasDatabaseConfig) {
    return createMysqlBackendRegistry();
  }

  return createBackendRegistry(); // インメモリフォールバック
}

const { app: honoApp, clearStorage: clearStorageFn } = resolveApp();
```

変更のポイントは2つ:

| 変更点 | 意図 |
|---|---|
| `hasDatabaseConfig` ガードを追加 | DB 設定がない環境で MySQL を呼ばないようにする |
| `resolveApp()` として関数に抽出 | テストから呼び出せるようにして TDD サイクルを回せるようにする |

`resolveApp()` を `export` したのは、後述するテストで直接インポートして「throw しないこと」を検証するため。

### `backend/src/infrastructure/migrate.ts` — DB 設定がなければスキップ

```typescript
// ✅ 修正後
async function migrate(): Promise<void> {
  if (!process.env.DATABASE_URL && !process.env.DB_USERNAME) {
    console.log('No database configured. Skipping migration.');
    return;                        // ← ここで早期リターン
  }

  const config = getMysqlConnectionConfig();
  // ... 既存のマイグレーション処理
}
```

DB 設定が存在しない場合はメッセージを出力して静かに終了する。
Render フリープランではこの早期リターンが実行されるため、ビルドフェーズのクラッシュもなくなった。

---

## TDD サイクル

### 🔴 RED — バグを再現するテストを先に書く

`backend/src/index.small.test.ts` を新規作成し、「本番環境かつ DB 設定なし」のシナリオで `resolveApp()` が throw しないことを表明した。

```typescript
import { describe, it, expect, afterEach, vi } from 'vitest';
import { resolveApp } from './index';

describe('resolveApp', () => {
  afterEach(() => {
    vi.unstubAllEnvs();
  });

  it('NODE_ENV=production かつ DB 設定なしでも throw しない', () => {
    vi.stubEnv('NODE_ENV', 'production');
    vi.stubEnv('DATABASE_URL', '');
    vi.stubEnv('DB_USERNAME', '');

    expect(() => resolveApp()).not.toThrow();
  });

  it('NODE_ENV=test では DB 設定の有無にかかわらずインメモリを返す', () => {
    vi.stubEnv('NODE_ENV', 'test');
    vi.stubEnv('DATABASE_URL', '');
    vi.stubEnv('DB_USERNAME', '');

    const result = resolveApp();

    expect(result.app).toBeDefined();
    expect(result.clearStorage).toBeTypeOf('function');
  });
});
```

修正前の `index.ts` でこのテストを実行すると、本番で起きていたのと**まったく同じエラー**が出て失敗することを確認した。

```
Error: DB_USERNAME and DB_PASSWORD environment variables are required
       when DATABASE_URL is not set
```

テストが本番障害を正確に再現できていることを確認してから、実装の修正に入った。

### 🟢 GREEN — 修正してテストを通す

`hasDatabaseConfig` ガードを追加したところ、2つのテストが両方パス。
既存の全 400 テストも引き続きパスし、型チェックおよび lint も clean だった。

---

## まとめ

| 項目 | 内容 |
|---|---|
| **症状** | Render フリープランでの本番起動が `DB_USERNAME and DB_PASSWORD are required` で即クラッシュ |
| **根本原因** | `index.ts` が `NODE_ENV !== 'test'` のとき DB 設定の有無にかかわらず MySQL レジストリを無条件呼び出していた |
| **発火条件** | Render フリープランは MySQL を提供しないため DB 環境変数が未設定 |
| **副次クラッシュ** | `render.yaml` のビルドステップ `migrate.mjs` も同じ `getMysqlConnectionConfig()` を呼んでビルドフェーズでもクラッシュ |
| **修正** | `hasDatabaseConfig` ガードで DB 設定がない場合はインメモリへフォールバック、`migrate.ts` には早期リターンを追加 |
| **TDD** | バグ再現テスト（RED）を先に書き、修正後に GREEN を確認 |

この修正の教訓は**「非テスト環境 ≠ MySQL が必ずある環境」** という前提の崩れにある。
インフラ構成が変わったとき（フリープラン移行、CI 環境、ローカル開発など）に「DB なし」ケースが現れうる。
`hasDatabaseConfig` のような単純なガードを一つ入れるだけで、異なるインフラ構成に対して同じバイナリが柔軟に動作できるようになる。
