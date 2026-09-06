# Wrangler生成Runtime型がBunの型チェックを壊した原因と分離方法

## 結論

`wrangler types`が生成したWorkers Runtime型を既存のBun用TypeScriptプロジェクトへそのまま混ぜると、グローバルWeb APIやバイナリ型の定義が変わり、Workerとは無関係な既存テストまで型エラーになることがある。

今回の解決方法は次の3点だった。

1. Bun用とWorkers用の`tsconfig`を分離する
2. Wranglerにはbinding型だけを生成させる
3. Nodeの暗号APIから得た値を`Buffer.from()`で明示的に変換する

## 発生した問題

### 症状

Workers用設定を追加して既定の`wrangler types`を実行した後、既存の型チェックで多数のエラーが発生した。

```text
TS18046: 'body' is of type 'unknown'.
TS2769: No overload matches this call.
```

既存Integrationテストが使う`Response.json()`の型が変わり、Worker対応と直接関係のないテストまで失敗した。Runtime型を`@cloudflare/workers-types`へ切り替えた後は、パスワード処理でも次のエラーが発生した。

```text
TS2554: Expected 0 arguments, but got 1.
```

エラー箇所は`randomBytes()`と`scryptSync()`の戻り値へ`.toString("hex")`を呼ぶ部分だった。

### 発生条件

- Environment: Windows、PowerShell
- Bun: 1.3.13
- Wrangler: 4.129.0
- TypeScript checker: `tsgo`
- Configuration: Bun型を使う既存backendへWorkers Runtime型を追加

### 影響

WorkerのSmallテストは成功したが、backend全体の型チェックを完了できなかった。

## 調査

### Worker実装の型を確認

```powershell
bun test src/worker.small.test.ts
```

Worker境界のテストは成功した。一方、全体の型チェックでは既存HTTP Integrationテストに`unknown`エラーが集中した。この差から、Worker処理ではなくプロジェクト全体へ読み込まれた型定義を調べた。

### Wrangler生成型の適用範囲を確認

既定の`wrangler types`は、bindingだけでなくWorkers Runtime全体の型を`worker-configuration.d.ts`へ生成する。そのファイルが既存`tsconfig.json`の探索範囲に入り、Bun用コードもCloudflare側のグローバル型で検査されていた。

既存`tsconfig.json`からWorker固有ファイルと生成型を除外し、`tsconfig.workers.json`だけでそれらを読み込むと、既存テストの`Response.json()`エラーは解消した。

### Runtime型全体が必要か確認

既定生成物は15,000行を超え、Wrangler自身が生成する末尾空白によって`git diff --check`にも失敗した。生成物を手作業で整形すると`wrangler types --check`との一致が失われる。

現在のCLIにはRuntime型を含めず、binding型だけを生成するオプションがある。

```powershell
wrangler types --include-runtime false
```

Runtime型は`@cloudflare/workers-types`から取得し、プロジェクト固有の`Env`だけをWrangler生成物にすると、生成ファイルは19行になった。

## 原因

実行環境ごとに意味が異なるグローバル型を1つのTypeScriptプロジェクトへ混在させたことが原因だった。

- Bun側はBunとNode互換型を前提にしていた
- Workers側はCloudflareのWeb APIとNode互換型を前提にした
- Wranglerの既定生成物がWorkers Runtime型全体を追加した
- 生成された`.d.ts`が既存backend全体へ適用された

暗号処理でも、Nodeの`Buffer`として暗黙に扱っていた戻り値がWorkers型のもとでは`Uint8Array`寄りに解釈され、`.toString("hex")`を受け付けなかった。

## 解決方法

### 1. TypeScriptプロジェクトを分離する

Bun用の`tsconfig.json`からWorker固有ファイルを除外し、Worker側では専用設定を使った。

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "types": ["bun-types", "@cloudflare/workers-types"]
  },
  "include": [
    "src/worker.ts",
    "src/worker.small.test.ts",
    "worker-configuration.d.ts"
  ],
  "exclude": []
}
```

型チェックはBun用とWorkers用の両方を実行する。

### 2. Binding型だけを生成する

```json
{
  "cf:typegen": "wrangler types --include-runtime false",
  "cf:typecheck": "wrangler types --include-runtime false --check"
}
```

`Env`を手書きせず、`wrangler.jsonc`をbinding型の正本にする。Runtime型はバージョン管理された`@cloudflare/workers-types`へ委ねる。

### 3. バイナリ型の境界を明示する

```ts
import { Buffer } from "node:buffer";

const salt = Buffer.from(randomBytes(16)).toString("hex");
const hash = Buffer.from(scryptSync(plain, salt, 64)).toString("hex");
```

既存のsaltとhashの保存形式は`hex:hex`のままで、挙動は変更していない。

## 動作確認

```text
bun run typecheck
  Bun用とWorkers用の両方が成功

bun run cf:typecheck
  生成binding型が最新

bun test
  131件成功

bunx wrangler deploy --dry-run
  Worker bundle成功
```

カバレッジはFunctions 85.85%、Lines 89.28%だった。実デプロイと本番Supabase疎通は未実施である。

## 再発防止

- Wrangler設定変更後は`cf:typegen`を実行する
- Bun用とWorkers用の両`tsconfig`を検査する
- 生成型の同期を`cf:typecheck`で確認する
- 実行環境固有の`.d.ts`を無関係なTypeScriptプロジェクトへ読み込ませない
- NodeとWeb APIの境界でバイナリ値を暗黙の`Buffer`として扱わない

## まとめ

- 症状: Worker対応後、既存Bunテストまで`unknown`や`.toString("hex")`の型エラーになった
- 原因: Workers Runtimeのグローバル型をBun用プロジェクト全体へ適用した
- 解決: `tsconfig`を分離し、binding型だけを生成し、バイナリ型を明示変換した
- 教訓: 複数ランタイムを扱う場合、コードだけでなく型の適用境界も分ける

## 参考資料

- [Cloudflare Workers TypeScript](https://developers.cloudflare.com/workers/languages/typescript/)
- [Wrangler commands](https://developers.cloudflare.com/workers/wrangler/commands/)
- 確認日: 2026年9月6日
