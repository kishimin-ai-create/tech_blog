# TDD ルールファイルに twada の推奨スタイルを追記した

## 対象読者

- TDD の実践スタイルをルールファイルとして明文化したい開発者
- `.github/rules/` 配下に設計方針を記録・整備している人
- 和田卓人（twada）氏の TDD スタイルをチームや AI エージェントに伝えたい方

---

## 背景

このリポジトリでは `.github/rules/test-driven-development.rules.md` が TDD の実践基準を定めており、OrchestratorAgent はこのルールに従って Red → Green → Refactor → Review のサイクルを制御する。

ただし、このファイルにはサイクルの手順は書かれていても、**なぜそのサイクルを踏むのか**という設計思想の根拠が欠けていた。エージェントが手順を守っても、その背景にある意図が伝わらなければ、手順の意味が形骸化するリスクがある。

今回の追記は、この設計思想の記録を補うことを目的としている。

---

## 追記した内容

`.github/rules/test-driven-development.rules.md` に 2 点を追記した。

### 1. 帰属の明記

ファイル冒頭に引用ブロックを追加し、このルールファイルが **twada（和田卓人）氏の推奨する TDD 実践スタイル** に基づいていることを明示した。

> TDD is not a testing technique but a **design technique**.

TDD は「テストを書く作業」ではなく「設計を進めるための手法」だという視点を、最初に読むエンジニアが即座に把握できるようにした。

### 2. `twada's Key Principles` セクションの新設

6 つの原則を追加した。

| 原則 | 概要 |
| :--- | :--- |
| **Baby steps** | 一歩が大きいと感じたらさらに細かく分割する |
| **Test list as a TODO list** | テストを書く前にシナリオを書き出し、一つずつ消化する |
| **Tests are living documentation** | テスト名は仕様書であり、読んで意図が伝わる名前にする |
| **Red confirms the test** | 必ず失敗（Red）を確認してから Green フェーズに進む |
| **Commit at each Green** | 全テストが通過した時点でコミットする習慣をつける |
| **Refactor only on Green** | テストが失敗している状態でリファクタリングしてはならない |

---

## TDD サイクルとの対応

これらの 6 原則は、OrchestratorAgent の TDD サイクル定義と直接対応している。

- **Red confirms the test** → Red Agent が生成したテストが必ず失敗する状態を確認してから Green フェーズへ移行する
- **Refactor only on Green** → Refactor Agent に移る前に、全テストがパスしていることを確認する
- **Commit at each Green** → Green 完了後に一度コミット可能な状態を記録する

特に **"Refactor only on Green"** は、エージェントが Refactor フェーズに入る前に必ずテストのパスを確認するというルールと直結しており、エージェント設計と原則が噛み合っている。

---

## 気をつけたいこと

- この追記は手順の変更ではなく、**既存の手順に設計思想の根拠を加えた**変更だ。実行結果はそのままで、読み手への説明が補強される。
- 今後 OrchestratorAgent や各 Sub-Agent のルールと原則が食い違う場合は、このファイルの原則を基準に整理するのが自然な流れになる。

---

## まとめ

今回の変更で TDD ルールファイルに twada の設計思想が明記された。

- ファイル冒頭に「TDD は設計手法である」という引用を置くことで、読み手が原則を即座に把握できるようになった
- 6 つの Key Principles が OrchestratorAgent の各フェーズ定義と対応づいており、エージェント運用の根拠として機能する

ルールファイルに「何をするか」だけでなく「なぜそうするか」を残しておくことで、エージェントも人間も同じ前提で動きやすくなる。
