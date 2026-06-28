# Storybook 10 でアドオンパッケージが core に統合された問題を修正した

## 対象読者

- Storybook を 8.x から 10.x へアップグレードした、またはアップグレードを検討しているフロントエンドエンジニア
- `package.json` のバージョン混在に起因するピア依存エラーやビルドエラーで困っている方

---

## 問題の背景

コードレビューで次の不整合が指摘された。

`frontend/package.json` の `devDependencies` を確認すると、Storybook のランタイム・フレームワーク系パッケージは **10.x** で揃っていた。

```json
"@storybook/nextjs": "^10.4.1",
"storybook": "^10.4.1"
```

一方、アドオン系の 3 パッケージだけ **8.x** が残ったままになっていた。

```json
"@storybook/addon-essentials": "^8.x.x",
"@storybook/addon-interactions": "^8.x.x",
"@storybook/test": "^8.x.x"
```

同じプロジェクト内で Storybook の 8.x と 10.x が混在している状態だ。

---

## 原因

Storybook 10 では、これら 3 つのパッケージが **`storybook` 本体に統合（absorbed）** された。

| パッケージ | Storybook 10 での扱い |
|---|---|
| `@storybook/addon-essentials` | `storybook` core に統合済み |
| `@storybook/addon-interactions` | `storybook` core に統合済み |
| `@storybook/test` | `storybook` core に統合済み |

これらは npm 上に **10.x 版が存在しない**。つまり `^10.0.0` の範囲で解決できるバージョンが npm レジストリに存在しない状態になる。

`bun install` がたまたま 8.x をキャッシュから引き続き使っていたか、あるいはバージョン指定が `^8.x.x` のまま残っていたかのいずれかで、表面上はインストールが通っていた。しかし、10.x のランタイムとピア依存の解決で不整合が生じ、ビルドや型チェック時に潜在的な問題を抱えた状態となっていた。

また、`.storybook/main.ts` の `addons` 配列にも `@storybook/addon-essentials` と `@storybook/addon-interactions` が登録されており、これらを Storybook 10 が起動時に解決しようとすると失敗する。

---

## 対応方針

core に統合されたパッケージは `package.json` から削除し、`addons` 配列からも取り除く。個別インストールは不要であり、`storybook` パッケージ本体が必要な機能を提供する。

---

## 実施した変更

### 1. `frontend/package.json` — 3 パッケージを削除

```diff
- "@storybook/addon-essentials": "^8.x.x",
- "@storybook/addon-interactions": "^8.x.x",
- "@storybook/test": "^8.x.x",
```

削除後の Storybook 関連 devDependencies は以下のとおりで、すべて 10.x で統一された。

```json
"@storybook/addon-vitest": "^10.4.1",
"@storybook/nextjs": "^10.4.1",
"eslint-plugin-storybook": "^10.4.1",
"storybook": "^10.4.1"
```

### 2. `frontend/.storybook/main.ts` — `addons` 配列から 2 エントリを削除

変更前：

```ts
addons: [
  "@storybook/addon-essentials",
  "@storybook/addon-interactions",
  "@storybook/addon-vitest",
],
```

変更後：

```ts
addons: [
  "@storybook/addon-vitest",
],
```

`@storybook/addon-vitest` は独立したパッケージとして 10.x 版が存在するため、そのまま残す。

### 3. `bun install` でロックファイルを更新

```bash
bun install
```

`bun.lock` から削除した 3 パッケージの解決エントリが除去され、依存グラフが整合した状態になった。

---

## 注意点

### `@storybook/test` の利用について

Storybook 10 では、テスト用ユーティリティ（`expect`、`userEvent` など）は `storybook/test` というサブパスエクスポートから import する。

```ts
// Storybook 8.x まで
import { expect } from "@storybook/test";

// Storybook 10 以降
import { expect } from "storybook/test";
```

既存の Story ファイルで `@storybook/test` を import していた場合は合わせて修正が必要だ。

### npm に 10.x が存在しないパッケージを誤って追加しないために

`npm info @storybook/addon-essentials versions` や `npm info @storybook/addon-interactions versions` を実行すると、最新が 8.x 系で止まっていることが確認できる。10.x 系のバージョンは存在しない。これがピア依存ミスマッチの根本原因だ。

---

## 検証結果

修正後、以下のコマンドがすべて正常終了（exit 0）したことを確認した。

```bash
bun run typecheck   # tsc --noEmit ✅
bun run lint        # eslint ✅
bun run test        # vitest run ✅
```

---

## まとめ

| 項目 | 内容 |
|---|---|
| **問題** | `@storybook/addon-essentials` / `@storybook/addon-interactions` / `@storybook/test` の 8.x パッケージが 10.x ランタイムと混在 |
| **根本原因** | Storybook 10 でこれら 3 パッケージが `storybook` core に統合され、npm 上に 10.x 版が存在しない |
| **修正** | `package.json` から 3 パッケージを削除、`main.ts` の `addons` 配列から 2 エントリを削除 |
| **確認** | typecheck / lint / test すべて pass |

Storybook のメジャーバージョンアップでは、アドオンの統廃合が行われることがある。バージョンを上げる際は、公式のマイグレーションガイドで各アドオンの状態（独立パッケージとして継続 / core 統合 / 廃止）を確認するのが確実だ。
