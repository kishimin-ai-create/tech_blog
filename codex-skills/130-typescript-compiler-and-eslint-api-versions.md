# TypeScriptのコンパイラーとESLintが使うAPIを別々に確認する

## はじめに

型チェックのコンパイラーが動作していても、ESLintが同じTypeScriptパッケージを利用できるとは限りませんでした。今回の環境では、backend・schemasのコンパイラーを維持しながら、共通Lint側にTypeScript 6を宣言して解消しました。

## 前提・環境

2026年10月21日、Windows、Bun 1.3.13、typescript-eslint 8.72系での観測です。TypeScriptの将来の版や、別のパッケージマネージャーへ結果を一般化しません。

## やってみた結果

共通のLintツールが解決するTypeScript APIと、各パッケージが起動するコンパイラーを分けました。

| 宣言場所 | TypeScript | 用途 |
| --- | --- | --- |
| app | `~6.0.3` | 共通ESLintツールが読むAPI |
| app/backend | `^7.0.2` | backendの型チェック |
| app/schemas | `^7.0.2` | schemasの型チェック |
| app/frontend | `~6.0.3` | frontendの型チェック |

インストール後、backend・schemasは7.0.2、frontendとLint APIは6.0.3であることを個別に確認しました。

## 実装・検証

### 必要な場所で依存を宣言する

変更は `f5c4845` に含まれます。ルートへTypeScript 6を置く場合でも、backendとschemasが必要とするコンパイラーは各devDependenciesに残しました。型チェックがルートの別版へ偶然依存する構成を避けるためです。

### lockfileと実際の解決先を検査する

`app` で `bun install --frozen-lockfile` を実行し、各パッケージのコンパイラー版とLintからのAPI解決を確認しました。

backend・schemasの型チェックは成功しました。ESLintも起動し、構文解析の互換性エラーではなく、ソース上のunsafe assignmentなどを報告する状態になりました。Lint違反そのものは残っています。

## つまずいたところ

途中で `typescript@npm:@typescript/typescript6` の互換パッケージ別名も試しました。しかし、このBun環境では `@typescript/old` を含む循環する解決に遭遇し、採用しませんでした。別名とoverrideを取り除き、通常の依存宣言へ戻しました。

これは今回のインストール結果です。互換パッケージがあらゆる環境で使えないという結論ではありません。

## 学んだこと

バージョン確認では、CLI名とAPI利用元を対応させる必要がありました。「TypeScriptを入れている」だけでは、どのコンパイラーとどのライブラリーがその版を使うかは説明できません。

## 根拠と制約

[共有ESLint導入コミット](https://github.com/kishimin/blog-wordcloud/commit/f5c4845)と `app/LINT.md`、各manifest、同期済みlockfile、版の照合結果が根拠です。今回の版の組み合わせで得られた記録であり、今後の互換性判断では再検証が必要です。
