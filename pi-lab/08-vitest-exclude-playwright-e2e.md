# VitestからPlaywright E2Eを確実に除外する

## 結論

VitestとPlaywrightを同じリポジトリで使う場合、`e2e/`をVitestの探索対象から明示的に除外する。プロジェクト別設定だけでなく、トップレベルの収集境界にも同じ除外を置く。

## 問題になる理由

どちらのランナーもTypeScriptのテストファイルを扱う。命名規則や`include`が広いと、VitestがPlaywright specを読み込み、Playwright用`test()`をVitest環境で評価する可能性がある。

ランナーの責務は次のように分けた。

| ディレクトリ | ランナー | 主な依存範囲 |
| --- | --- | --- |
| `src/**/*.test.ts(x)` | Vitest | 関数、React、jsdom |
| Storybook stories | Vitest Browser Mode | 実ブラウザ上のStorybook |
| `e2e/**/*.spec.ts` | Playwright Test | アプリ全体とブラウザ |

## Vitestのデフォルト除外を保持する

独自`exclude`でデフォルト値を失わないよう、`configDefaults.exclude`を展開した。

```ts
import { configDefaults } from "vitest/config";

const defaultAndE2EExcludes = [
  ...configDefaults.exclude,
  "**/e2e/**",
];
```

独自パターンだけを設定すると、`node_modules`などVitestが通常除外する対象を意図せず収集する可能性がある。既定値へ追加する形なら、標準境界を維持できる。

## トップレベルとプロジェクトへ適用する

```ts
export default mergeConfig(
  viteConfig,
  defineConfig({
    test: {
      exclude: defaultAndE2EExcludes,
      projects: [
        {
          extends: true,
          test: {
            environment: "jsdom",
            exclude: [
              ...defaultAndE2EExcludes,
              "**/*.stories.{ts,tsx,mdx}",
              "**/.storybook/**",
            ],
          },
        },
        // Storybook Browser Mode project
      ],
    },
  }),
);
```

トップレベルの`exclude`は全体の収集境界を表し、jsdomプロジェクト側ではstoryも追加で除外する。設定を`vite.config.ts`から`vitest.config.ts`へ分離したことで、Vite本体とテストランナーの責務も明確になった。

## 型検査対象を忘れない

新しく作った`vitest.config.ts`を`tsconfig.node.json`へ追加した。

```json
{
  "include": [
    "vite.config.ts",
    "vitest.config.ts",
    "playwright.config.ts"
  ]
}
```

設定ファイルも実行コードである。型検査から外すと、importや設定APIの変更をCIまで見逃しやすい。

## 検証

コミット`45fda26`で設定を分離し、E2E除外を追加した。Playwright specはPlaywrightだけが、UnitとStorybookはVitestだけが担当する状態になった。

## 制約

将来E2Eディレクトリを変更した場合は、Playwrightの`testDir`とVitestの`exclude`を同時に見直す必要がある。ファイル名だけへ依存する除外より、所有ディレクトリで境界を表す方が変更を追跡しやすい。

## 参考資料

- [Vitest: exclude configuration](https://vitest.dev/config/exclude)
- 根拠コミット: `45fda26`
- 確認日: 2026-08-06
