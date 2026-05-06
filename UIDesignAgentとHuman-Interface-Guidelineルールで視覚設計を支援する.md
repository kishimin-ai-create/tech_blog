# UIDesignAgentとHuman-Interface-Guidelineルールで視覚設計を支援する

## 対象読者

- フロントエンドの UI 品質を上げたいが「ロジックを壊さずスタイルだけ直したい」と思っているエンジニア
- Tailwind CSS v4 + React のプロジェクトで一貫したデザインを維持したい人
- Copilot エージェントを UI 改善に活用したい人

## この記事が扱う範囲

- **扱う**: UIDesignAgent の仕様・制約・ワークフロー、human-interface-guideline.rules.md の位置付け
- **扱わない**: Tailwind CSS の基本的な使い方、React のコンポーネント設計

---

## 背景

バックエンドの TDD サイクルや Clean Architecture の整備が進む中、フロントエンドには「UI のビジュアル品質を改善する専用エージェント」が存在していなかった。

機能実装を担う Red/Green/Refactor のエージェント群は、スタイルや見た目の改善を主眼に置いていない。
UI を改善しようとすると「ロジックを変えずにスタイルだけ直す」という制約を誰も明文化していなかったため、
意図せずコンポーネントのロジックや props 定義に手が入るリスクがあった。

---

## 追加されたもの

今回のコミットで以下の2ファイルが追加された。

```
.github/agents/UIDesignAgent.agent.md       # UIデザイン専任エージェント定義
.github/rules/human-interface-guideline.rules.md  # UI設計原則ルールファイル（100原則）
```

---

## UIDesignAgent の役割

UIDesignAgent は **「スタイルのみを担当する視覚デザインの専門家」** として定義されている。

> You are a frontend design specialist focused on **visual polish, consistency, and accessibility**.  
> Your purpose is to improve the look and feel of the application using the existing tech stack —  
> React 19, TypeScript, and Tailwind CSS v4 — **without changing component logic or breaking tests**.

### 入力

エージェントが受け取れる入力の例：

| 入力の種類 | 例 |
|---|---|
| コンポーネントのパス | `frontend/src/features/todo/TodoItem.tsx` |
| デザインの参照 | `docs/design/` 以下の仕様書やスクリーンショット |
| スコープキーワード | `"全体の余白を整えて"` / `"モバイル対応して"` |

スコープが指定されない場合は、`frontend/src` 全体を監査して改善点を適用する。

### 出力

UIDesignAgent が完了時に必ず届けるもの：

1. Tailwind クラスが更新されたコンポーネントファイル
2. 更新または新規作成された Storybook ストーリー（`*.stories.tsx`）
3. `cd frontend && npm run build` の通過
4. `cd frontend && npm run lint` の通過
5. 変更ファイルと改善点のサマリー

---

## 外せないコアルール

UIDesignAgent には「スタイルのみ」という絶対的な制約が定義されている。

### ✅ やること

- Tailwind CSS v4 のユーティリティクラスのみで改善する
- モバイルファーストで `sm:` ブレークポイントから設計する
- WCAG AA 基準（テキスト 4.5:1、UI 要素 3:1）のカラーコントラストを保つ
- すべてのインタラクティブ要素に `focus-visible:` クラスを付与する
- コンポーネント変更後は必ず Storybook ストーリーを追加/更新する

### ❌ やってはいけないこと

- イベントハンドラ・state・props インターフェースの変更
- テストの assertion やテスト構造の変更
- `style={{}}` によるインラインスタイルの追加
- 新しい npm パッケージのインストール
- バックエンドコードへの変更
- `aria-label`・`alt`・`role` などアクセシビリティ属性の削除

---

## Tailwind CSS v4 のコーディング例

エージェント定義内には実際のコードスニペットが含まれている。

```tsx
// ✅ インタラクティブ状態を持つボタン
<button className="
  bg-blue-600 text-white px-4 py-2 rounded-md font-medium
  hover:bg-blue-700 active:bg-blue-800
  focus-visible:outline focus-visible:outline-2 focus-visible:outline-blue-500
  disabled:opacity-50 disabled:cursor-not-allowed
  transition-colors duration-150
">

// ✅ 統一されたカードスタイル
<div className="rounded-xl border border-gray-200 bg-white p-5 shadow-sm
                hover:shadow-md transition-shadow duration-200">
```

---

## human-interface-guideline.rules.md とは

100 の UI 設計原則をカテゴリ別にまとめたルールファイル。
UIDesignAgent の Governing Rules テーブルでプライマリ参照先として位置づけられている。

```
| .github/rules/human-interface-guideline.rules.md | UI/UX design principles — primary reference |
```

原則はカテゴリに分類されており、代表的なものは次のとおり。

| カテゴリ | 原則の例 |
|---|---|
| Core Principles | Simplicity, Ease of Use, Mental Model, Signifiers |
| Cognitive Load | Hick's Law, Fitts' Law, Error Prevention |
| Interaction Design | State Visibility, Noun-Verb Order |
| Layout & Visual | Tone & Manner Consistency, Proximity of Tools |
| Forms & Input | Good Defaults, Structured Input Flow |
| Animation & Feedback | Fast Response (0.1秒以内), Local Feedback |
| Mobile & System | Touch Target Size (最低7mm), Hierarchical Design for Mobile |

これらはコードと直接対応するルールファイルであり、UIDesignAgent が改善案を検討する際の判断基準となる。

---

## 調査ワークフロー

UIDesignAgent が改善作業を行う際の手順は4段階。

### Step 1: デザインコンテキストを読む

`docs/design/` 配下のカラーパレット・スペーシングスケール・タイポグラフィルールを確認する。
ドキュメントがない場合は既存コードから推測する。

### Step 2: 既存コンポーネントを監査する

`frontend/src/` の全 `.tsx` ファイルを対象に次を確認する。

- スペーシングの不統一（`p-2` と `p-3` が混在するなど）
- カラーの不統一（`blue-500` と `blue-600` が混在するなど）
- `focus:` スタイルの欠落
- レスポンシブ対応の漏れ
- ARIA 属性の欠落

### Step 3: 変更を適用する

同じ機能配下のコンポーネントをバッチで編集する。
変更は `className` 文字列と表示専用のマークアップ（レイアウト用 `<div>` のラップなど）のみ。

### Step 4: 検証する

```bash
cd frontend
npm run build  # 必ず通過させる
npm run lint   # 0 エラーであること
```

---

## まとめ

UIDesignAgent の追加により、「UI をきれいにしたい」という要求を受け取れる専門エージェントが生まれた。
`human-interface-guideline.rules.md` の 100 原則がその判断の拠り所となり、  
「スタイルのみ・ロジックは変更しない」という明確な制約が既存のテストやロジックを守る。

ロジック層の品質管理（TDD サイクル）と視覚層の品質管理（UIDesignAgent）が役割分担されることで、
どちらも妥協せずに進められる体制が整った。
