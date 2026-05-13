# Windows で生成した package-lock.json が Linux CI で `npm ci` を壊す問題を修正した

## エラー概要

バックエンドの GitHub Actions ワークフロー（ubuntu-latest）で `npm ci` が失敗していました。エラーは `npm ci` が `package.json` と `package-lock.json` の不一致を検出して中断するものです。

ローカル（Windows）では `npm install` も `npm ci` も正常に動いていたため、CI のみで発覚した問題でした。

## 原因

`backend/package-lock.json` が Windows 環境で生成されており、Linux 向けの **プラットフォーム固有エントリ** が不足していました。

### esbuild のプラットフォーム固有パッケージ

`esbuild` は実行バイナリをプラットフォームごとに分割したオプショナルパッケージとして配布しています。

| パッケージ名 | 対象 |
|---|---|
| `@esbuild/win32-x64` | Windows x64 |
| `@esbuild/linux-x64` | Linux x64 |
| `@esbuild/darwin-arm64` | macOS Apple Silicon |
| … | … |

Windows で `npm install` を実行すると、`package-lock.json` には Windows 用エントリのみが記録されます。このロックファイルを Linux の CI で `npm ci` に渡すと、Linux 用エントリが存在しないため整合性チェックに失敗します。

`npm ci` は `package-lock.json` と `package.json` が **完全に一致している** ことを前提とするため、プラットフォームエントリが欠けていると問答無用でエラーになります。

### 変更規模

```
backend/package-lock.json | 105 +++++++++++++++++++++++++++++++++--
 1 file changed, 86 insertions(+), 19 deletions(-)
```

Linux・macOS などのエントリが 86 行追加されており、これが欠落していたプラットフォーム固有エントリに相当します。

## 修正

`backend/` ディレクトリ内で `npm install` を再実行し、`package-lock.json` を再生成しました。

```bash
cd backend
npm install
```

`npm install` はクロスプラットフォーム対応の形式でロックファイルを生成し直すため、Linux・macOS を含む全プラットフォーム向けの esbuild エントリが追記されます。再生成後、以下の確認をすべて通過しました。

- `npm run typecheck` ✅
- `npm run lint` ✅
- `npm run build` ✅

## 気をつけること

### `npm install` と `npm ci` の使い分け

| コマンド | 用途 | lock ファイル |
|---|---|---|
| `npm install` | 開発中の依存追加・更新 | 更新される |
| `npm ci` | CI / 本番デプロイ | 変更不可（不一致でエラー） |

CI では `npm ci` を使うのが正しい選択ですが、その前提として **ロックファイルが CI 環境でも有効** である必要があります。

### 複数 OS で開発するプロジェクトへの注意

チームメンバーが異なる OS を使う場合、`package-lock.json` の生成 OS によってプラットフォームエントリが変わります。現在の npm（v7 以降）はマルチプラットフォーム対応の lock を生成できますが、生成環境によっては一部エントリが省かれることがあります。

対策として、次のいずれかを検討できます。

- CI で最初に `npm install` を実行してロックファイルを更新してから処理する（ただし再現性が下がる）
- ロックファイルの再生成を別途 PR として管理し、クロスプラットフォーム環境（Linux など）で生成する運用にする

本リポジトリでは Windows で再生成したロックファイルをコミットし直す方式で対処しました。

## まとめ

| 観点 | 内容 |
|---|---|
| 根本原因 | `package-lock.json` が Windows 環境で生成され、Linux 用 esbuild エントリが欠落していた |
| 修正方法 | `backend/` で `npm install` を再実行してロックファイルを再生成 |
| 変更規模 | `package-lock.json` に 86 行追加・19 行変更 |
| 再発防止 | 新しいネイティブバイナリを持つパッケージを追加した際は、CI 環境（Linux）でのロックファイル有効性を確認する |

`npm ci` は「ロックファイルと完全に一致した状態」を保証するためのコマンドです。その厳格さが CI の再現性を担保する一方で、ロックファイル自体がクロスプラットフォーム対応になっていないと CI だけで失敗する落とし穴があります。
