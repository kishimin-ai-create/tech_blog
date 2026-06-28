# `import { create } from "axios"` は無効 — axios.create の正しいインポート方法

## エラー概要

Orval が生成した API クライアントのカスタムインスタンスファイル (`frontend/app/api/mutator/custom-instance.ts`) に、次のコードが含まれていた。

```ts
// ❌ 誤り
import { create } from "axios";

const axiosInstance = create({ baseURL: BACKEND_URL });
```

このコードは **TypeScript の型検査でエラーになり、ビルドも失敗する**。

---

## 原因

axios は `create` を **名前付きエクスポート (named export) として公開していない**。

axios のパッケージ構造を確認すると、`create` メソッドはデフォルトエクスポートされたオブジェクト（`AxiosStatic`）のプロパティとして定義されている。

```ts
// axios の型定義（抜粋）
interface AxiosStatic extends AxiosInstance {
  create(config?: CreateAxiosDefaults): AxiosInstance;
  // ...
}

declare const axios: AxiosStatic;
export default axios;  // ← デフォルトエクスポートのみ
```

`{ create }` のような分割代入インポートは **ES モジュールの名前付きエクスポートを参照する構文**であり、axios はそれを提供していない。そのため TypeScript は「`create` というエクスポートが存在しない」と判定し、コンパイルエラーを発生させる。

---

## 修正

デフォルトインポートに変更し、`axios.create(...)` として呼び出す。

```ts
// ✅ 正しい
import axios from "axios";
import type { AxiosRequestConfig } from "axios";

const BACKEND_URL = process.env.NEXT_PUBLIC_BACKEND_URL ?? "http://localhost:3000";

const axiosInstance = axios.create({ baseURL: BACKEND_URL });

/**
 * Custom Axios instance used by Orval-generated API clients.
 * Automatically injects the backend base URL and handles response unwrapping.
 */
export const customInstance = async <T>(config: AxiosRequestConfig): Promise<T> => {
  const response = await axiosInstance<T>(config);
  return response.data;
};

export type ErrorType<Error> = Error;
```

変更点は 2 箇所のみ。

| 変更前 | 変更後 |
|---|---|
| `import { create } from "axios"` | `import axios from "axios"` |
| `create({ baseURL: ... })` | `axios.create({ baseURL: ... })` |

---

## 検証

`frontend/` ディレクトリで型検査を実行し、エラーがないことを確認した。

```bash
bun run typecheck
# exit code 0
```

---

## まとめ

| 項目 | 内容 |
|---|---|
| **対象ファイル** | `frontend/app/api/mutator/custom-instance.ts` |
| **症状** | TypeScript 型エラー・ランタイムビルド失敗 |
| **根本原因** | axios は `create` を名前付きエクスポートしていない |
| **修正方法** | デフォルトインポートに変更し `axios.create()` を使う |

### ポイント

- axios に限らず、**ライブラリのエクスポート方式（default か named か）は型定義を一読して確認する**習慣が有効。
- Orval などのコードジェネレーターが吐き出したカスタムインスタンスのテンプレートをそのまま使う場合も、インポート文が実際のパッケージ仕様と合致しているかを確認する必要がある。
- `bun run typecheck` を CI に組み込んでおくことで、同種のミスを早期に検出できる。
