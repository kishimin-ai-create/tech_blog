# `bun test` はテストファイルが0件だとexit 1する — CIガードで回避する

## はじめに

バックエンドにはまだテストファイルが1件も存在しないため、`bun run test:small` を実行するとパターンにマッチするファイルが 0 件となり、`bun test` は **exit code 1** で終了する。結果、CI は全ての backend テストステップで失敗していた。

## 対象読者

- Bun をテストランナーとして使っている GitHub Actions ユーザー
- バックエンドにまだテストファイルがなく、CI が空振りで落ちて困っている人
- `|| true` や `continue-on-error` を使わずに安全にスキップしたいエンジニア

---

## 背景

このリポジトリのバックエンドは Bun で動いており、テストコマンドは次のように定義されている。

```json
// backend/package.json（抜粋）
"test:small":  "bun test --testPathPattern='\\.small\\.test\\.'",
"test:medium": "bun test --testPathPattern='\\.medium\\.test\\.'",
"test:large":  "bun test --testPathPattern='\\.large\\.test\\.'"
```

GitHub Actions ワークフローでは、この 3 コマンドをそれぞれのステップで呼んでいた。

```yaml
# 修正前（ci.yml / ci-pr.yml / ci-nightly.yml 共通）
- name: Run small tests
  run: bun run test:small
```

バックエンドにはまだテストファイルが1件も存在しないため、`bun run test:small` を実行するとパターンにマッチするファイルが 0 件となり、`bun test` は **exit code 1** で終了する。結果、CI は全ての backend テストステップで失敗していた。

---

## 根本原因：`bun test` に `--passWithNoTests` がない

Vitest には `passWithNoTests: true` オプションがあり、テストファイルが 0 件でも exit 0 で終了できる。

一方 `bun test` にはこれに相当するフラグが**存在しない**。テストファイルが見つからない場合、コマンドは必ず exit 1 を返す。

| ランナー | 0件時のデフォルト動作 | 0件でも成功扱いにする方法 |
|----------|----------------------|--------------------------|
| Vitest   | exit 0（デフォルトで安全） | `passWithNoTests: true`（明示的に設定） |
| bun test | **exit 1（失敗）**  | **なし** |

---

## 採用しなかった解決策

### `|| true` を付ける

```yaml
run: bun run test:small || true
```

コマンドが何らかの理由で本当に失敗しても exit 0 になる。**本物のテスト失敗を隠す**ので採用しなかった。

### `continue-on-error: true` を使う

```yaml
- name: Run small tests
  run: bun run test:small
  continue-on-error: true
```

ステップが失敗してもジョブを継続できるが、ステップ自体は「失敗」と記録される。テストファイルが追加されたあとも失敗が見えなくなるリスクがある。採用しなかった。

---

## 採用した解決策：実行前にファイル数をカウントするシェルガード

```yaml
- name: Run small tests
  run: |
    count=$(find src -name "*.small.test.*" 2>/dev/null | wc -l)
    if [ "$count" -gt 0 ]; then
      bun run test:small
    else
      echo "No backend small test files found — skipping"
    fi
```

### ガードの動作

1. `find src -name "*.small.test.*"` で `src/` 配下の対象ファイルを列挙する
2. `wc -l` で件数をカウントする
3. 1件以上あれば `bun run test:small` を実行し、その終了コードがジョブに伝播する
4. 0件なら skip メッセージを出力して **exit 0** で終わる

`2>/dev/null` は `src/` ディレクトリが存在しない場合でも `find` のエラー出力を抑制するためのものだ。

### テストファイルが増えたら自動で有効になる

このガードは「テストが増えたら `count` が 1 以上になり、自動的に `bun run test:small` が実行される」仕組みだ。ワークフローを手動で変更する必要はない。

---

## 変更対象ワークフロー

同じガードを 3 つのワークフローファイルのバックエンドジョブに適用した。

| ファイル | バックエンドで実行するテストサイズ |
|----------|------------------------------------|
| `ci.yml` | small のみ |
| `ci-pr.yml` | small + medium |
| `ci-nightly.yml` | small + medium + large |

`ci-pr.yml` と `ci-nightly.yml` では medium / large のステップも同じパターンでガードしている。

```yaml
# medium の例（ci-pr.yml）
- name: Run medium tests
  run: |
    count=$(find src -name "*.medium.test.*" 2>/dev/null | wc -l)
    if [ "$count" -gt 0 ]; then
      bun run test:medium
    else
      echo "No backend medium test files found — skipping"
    fi
```

---

## フロントエンド側は変更不要

フロントエンドのテストランナーは Vitest であり、`vitest.config.ts` に `passWithNoTests: true` が設定済みだ。テストファイルが 0 件でも exit 0 で終わるため、同様のガードは不要だった。

```ts
// frontend/vitest.config.ts
test: {
  passWithNoTests: true,  // ← 0件でも成功
  // ...
}
```

---

## まとめ

| 観点 | 内容 |
|------|------|
| 問題 | `bun test` はマッチするファイルが 0 件だと exit 1 を返す |
| 採用しなかった対処 | `\|\| true`、`continue-on-error: true` — 本物の失敗を隠すリスクがある |
| 採用した対処 | `find` でファイル数をカウントし、0 件なら skip するシェルガード |
| 自動回復 | テストファイルが追加されると count > 0 になり、自動的に実行される |
| 対象ワークフロー | `ci.yml`、`ci-pr.yml`、`ci-nightly.yml` のバックエンドジョブ |

`bun test` を使う場合は「ファイルが 0 件でも成功扱いにするオプションがない」という制約を念頭に置いておきたい。CI の段階が増えてテストの分類が増えるほど、このガードパターンは繰り返し役立つ。
