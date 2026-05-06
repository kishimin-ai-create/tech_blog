# カバレッジ閾値を復元してCIカバレッジ契約を修復した

## 対象読者

- TypeScript + Vitest でテスト環境を整備しているバックエンド・フロントエンドエンジニア
- `coverage.thresholds` の役割と、それを無効化することのトレードオフを理解したい人
- CI でのカバレッジ強制をどのタイミングで有効にすべきか迷っている人

---

## 問題の背景

TDD Todo App では Vitest（バックエンド）と Vite + Vitest（フロントエンド）の 3 つの設定ファイルで
カバレッジを計測している。

| 設定ファイル | 役割 |
|---|---|
| `backend/vitest.unit.config.ts` | バックエンドのユニットテスト用カバレッジ |
| `backend/vitest.integration.config.ts` | バックエンドの統合テスト用カバレッジ |
| `frontend/vite.config.ts` | フロントエンドのコンポーネント・ユニットテスト用カバレッジ |

これら 3 ファイルには `coverage.thresholds` として以下の閾値が設定されていた。

```ts
thresholds: {
  lines: 80,
  branches: 75,
  functions: 80,
  statements: 80,
},
```

---

## 経緯：閾値の削除と、その影響

以前のコミット（`984486d`）で、CI が常に失敗する問題を解消するために
**3 ファイルからこの `thresholds` ブロックがまるごと削除された**。

当時の理由は正当だった。コードベースのテストカバレッジが閾値に届いておらず、
Vitest がカバレッジ計測後に非ゼロで終了し、GitHub Actions ワークフロー末尾の
「カバレッジ失敗ゲート」が毎回発火していた。

```yaml
# test-coverage.yml の一部
- name: Fail when coverage failed
  if: >
    steps.frontend_coverage.outcome == 'failure' ||
    steps.backend_unit_coverage.outcome == 'failure' ||
    steps.backend_integration_coverage.outcome == 'failure'
  run: exit 1
```

しかし閾値を削除したことで別の問題が生じた。
**リポジトリが「カバレッジ閾値を強制する」というカバレッジ契約を失ってしまった**。
CI はグリーンになったが、カバレッジが何パーセントまで落ちても検知できなくなった。

---

## 修正内容

コミット `9622ec5`（バックエンド）と `1aab1ec`（フロントエンド）で、
`thresholds` ブロックが 3 ファイルへ復元された。

### バックエンド：`vitest.unit.config.ts`

```diff
       exclude: [
         'node_modules/',
         'dist/',
         'coverage/',
         '**/*.config.ts',
-        '**/tests/**'
-      ]
+        '**/tests/**',
+      ],
+      thresholds: {
+        lines: 80,
+        branches: 75,
+        functions: 80,
+        statements: 80,
+      },
     }
```

### バックエンド：`vitest.integration.config.ts`

```diff
       exclude: [
         'node_modules/',
         'dist/',
         'coverage/',
         '**/*.config.ts',
-        '**/tests/**'
-      ]
+        '**/tests/**',
+      ],
+      thresholds: {
+        lines: 80,
+        branches: 75,
+        functions: 80,
+        statements: 80,
+      },
     }
```

### フロントエンド：`vite.config.ts`

```diff
       exclude: [
         'node_modules/',
         'dist/',
         'coverage/',
         '**/*.config.ts',
         '**/test/**',
         '**/*.spec.ts',
-        '**/*.test.ts'
-      ]
+        '**/*.test.ts',
+      ],
+      thresholds: {
+        lines: 80,
+        branches: 75,
+        functions: 80,
+        statements: 80,
+      },
     }
```

3 ファイルとも変更内容は同一パターンで、差分は次の 2 点に集約される。

1. **`thresholds` ブロックの追加** — `exclude` 配列の直後に配置
2. **`exclude` 末尾項目へのトレーリングカンマ追加** — `**/tests/**` および `**/*.test.ts` 末尾に `,` を追記

> **配置について**: 以前の設定では `thresholds` は `include`/`exclude` より前にあった。
> 今回の復元では `exclude` の直後に置かれており、カバレッジ対象ファイルの定義が
> 先に確定した状態で閾値が記述される順序になっている。

---

## なぜ今回は閾値を復元できたか

前回の削除時は「テストが十分に書かれていない」ことが原因で閾値を達成できなかった。
今回の復元が可能になったのは、その後の実装でテストが十分に揃ったためである。

修正後の実行結果：

| スコープ | 結果 |
|---|---|
| バックエンド | 391 / 391 テスト通過 |
| フロントエンド | 147 / 147 テスト通過 |
| `npm run lint` | エラー 0 件 |
| `npm run typecheck` | エラー 0 件 |
| `npm run test` | エラー 0 件 |

カバレッジが閾値（lines: 80, branches: 75, functions/statements: 80）を満たした状態で
Vitest が正常終了するため、CI の失敗ゲートも発火しない。

---

## 閾値の設計上の注意点

`coverage.thresholds` は Vitest の v8 プロバイダーにとって「**テストが通過する条件**」の一部として扱われる。
閾値を下回ると Vitest プロセスが非ゼロで終了するため、CI ワークフローとの組み合わせに注意が必要だ。

```
閾値未達 → Vitest が exit non-zero
      ↓
continue-on-error: true で後続ステップは継続
      ↓
step.outcome = 'failure' が確定
      ↓
最終ゲート（exit 1）が発火 → ワークフロー失敗
```

`continue-on-error: true` を使う「後でまとめて判定」パターンを採用している場合、
閾値は設定するだけで自動的にゲートとして機能する。
一方で、テストが育っていない段階で高い閾値を設定すると CI が常に失敗する状態になる。

**推奨タイミング**: 主要ユースケースのユニット・統合テストが揃い、実測カバレッジが
安定して閾値を上回るようになった段階で `thresholds` を導入するのが適切である。

---

## まとめ

- `984486d` での閾値削除は CI を緑に戻したが、カバレッジ強制の契約を失うトレードオフがあった
- テストが十分に揃った段階で `9622ec5` / `1aab1ec` が閾値を復元し、カバレッジ契約を修復した
- 復元後の閾値は `exclude` 配列の直後に配置され、全 3 設定ファイルで統一された
- バックエンド 391 件・フロントエンド 147 件すべてのテストが通過し、lint・typecheck も完全にクリーン
- `coverage.thresholds` は「カバレッジを計測する設定」ではなく「カバレッジを強制する設定」であり、適切なタイミングで有効化することが重要
