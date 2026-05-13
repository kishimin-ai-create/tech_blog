# Devlog 2026-05-10: 幻のバグ—— Chrome オートフィル、リポジトリ移行、Render デプロイの謎

**日付**: 2026-05-10  
**スタック**: React + TypeScript + Vite + Jotai + Hono + Vitest + Render
**コミット**: `bc16cad` — `fix: add autoComplete attributes to auth forms to prevent browser autofill`

---

## 概要

今日のセッションで3つの問題が解決されました：

1. **Sign Up フォームがナビゲーション後も値を保持するというバグ報告** — 実際の原因はアプリケーションの状態ではなく、Chrome のオートフィルでした
2. **GitHub リポジトリ移行** — 個人アカウントから organization アカウントへの移行と、CI・デプロイへの影響
3. **Render デプロイの混乱** — ログに `yarn start` の失敗が出ていたが、バックエンドは実際には正常に起動していた

---

## 1. 幻のバグ（Chrome がそれを現実にした）

### バグ報告

> 「Sign Up フォームに値を入力してLogin に移動し、また Sign Up に戻ると——前に入力した値がフィールドに残っている」

典型的なフォーム状態の漏れ。グローバルスコープの `useState`、あるいは条件付きレンダリングコンポーネントの `key` prop が欠けているか。調査を始めました。

### 調査：すべての証拠が「バグなし」を指していた

フォームの状態は `useAuthForm.ts` の中にすべて収まっています：

```ts
// frontend/src/features/auth/hooks/useAuthForm.ts
export function useAuthForm({ endpoint, fallbackErrorMessage, onSuccess }) {
  const [email, setEmail]       = useState('')
  const [password, setPassword] = useState('')
  // ...
}
```

ローカルの `useState` ——Jotai の atom もグローバルスコープもありません。この状態はコンポーネントがアンマウントされると消えます。

次に `App.tsx` を確認しました：

```tsx
// frontend/src/App.tsx
if (!auth) {
  if (currentPage.name === 'signup') return <SignupPage key="signup" />
  if (currentPage.name === 'login')  return <LoginPage  key="login"  />
  return <LandingPage />
}
```

ナビゲーションのたびにクリーンなマウントを保証する2つの仕組み：

- `SignupPage` と `LoginPage` は **異なるコンポーネント型** ——型が変わるたびに React が古いものをアンマウントして新しいインスタンスをマウントします
- 両方に明示的な `key` prop（`key="signup"`、`key="login"`）があります ——前のコミット（`ca13d16`）で追加された防御的な措置で、2つのページが1つの共有コンポーネントにリファクタリングされた場合でも状態の独立性を保証します

180のテストを実行するとすべてパス。`App.test.tsx` の回帰テストも含め、正確に `signup → login → signup` のナビゲーションフローを実行してステップごとにフィールドが空であることをアサートしています。

**コードレビューの結論：アプリケーションの状態は正しく動作している。**

### 本当の根本原因：Chrome の認証情報マネージャー

実際の犯人は **React の再マウント時にブラウザのオートフィルが発火すること** でした。

この SPA で、Chrome がページ遷移時に何をするかを説明します：

1. ユーザーが Sign Up ページにメールアドレスとパスワードを入力
2. Chrome の認証情報マネージャーが新しい `<input type="email">` と `<input type="password">` 要素を認識
3. ユーザーがLoginに移動——React が `SignupPage` をアンマウントして `LoginPage` をマウント
4. ユーザーが再び Sign Up に戻る——React が `LoginPage` をアンマウントして `SignupPage` をマウント
5. **Chrome はページリロードなしに同じ URL で新しい `<input>` 要素が出現したと認識**
6. Chrome が制御された入力に `onChange` イベントを発火させて、以前に入力（または保存）された値を注入
7. React がこれらの合成イベントを処理して `useState` を更新

入力は制御されている（`value={email}` + `onChange`）ため、Chrome のオートフィルは DOM を直接変更するのではなく `onChange` を dispatch することで動作します。React の観点からは実際のユーザー入力と区別がつきません。マウント時の `useState` 値は確かにゼロでしたが、マウント直後にオートフィルがそこに書き込んでいたのです。

これはよく知られた SPA 特有の動作です：従来のページナビゲーションはブラウザの状態をフラッシュしますが、同じ URL での React の再マウントはそうしません。

### 修正: `autoComplete` 属性

修正はシンプル——フォームごとに2つの属性を追加するだけ——しかし値の選択が重要です。

**`SignupPage.tsx`**

```tsx
<input
  id="email"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  autoComplete="off"          // ← 追加
/>
<input
  id="password"
  type="password"
  value={password}
  onChange={(e) => setPassword(e.target.value)}
  autoComplete="new-password" // ← 追加
/>
```

**`LoginPage.tsx`**

```tsx
<input
  id="email"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  autoComplete="username"         // ← 追加
/>
<input
  id="password"
  type="password"
  value={password}
  onChange={(e) => setPassword(e.target.value)}
  autoComplete="current-password" // ← 追加
/>
```

### この値を選んだ理由

| フィールド | 値 | 理由 |
|---|---|---|
| SignupPage メール | `"off"` | 新規アカウントフォームのオートフィルを抑制——保存された認証情報をここに事前入力すべきではない |
| SignupPage パスワード | `"new-password"` | Chrome にこれがアカウント作成フォームであることを伝える。Chrome は新しい認証情報の保存を提案するが、**以前に保存したものを注入しない** |
| LoginPage メール | `"username"` | ログインメールの正しいセマンティック値。ブラウザのパスワードマネージャーに保存された認証情報をマッチングするよう伝える |
| LoginPage パスワード | `"current-password"` | `username` と組み合わせて、ブラウザのパスワードマネージャーがログインページでオートフィルできるようにする——これが意図されたUX |

重要な気づき：`new-password` と `current-password` は Chrome が「アカウント作成」と「サインイン」を区別するために使うセマンティックシグナルです。正しく設定することで、サインアップ時のオートフィル問題を解決しながら、ログイン時の意図されたオートフィル動作を維持できます。

### 結果

- 認証関連の28/28テストが修正後もパス
- 全180テストがパス
- テストの変更は不要：テスト環境（Vitest + jsdom）はブラウザのオートフィルをエミュレートしないため、既存の回帰テストはすでに基礎となる React の状態動作を正しくカバーしていた

### 教訓

180のユニット/統合テストがすべてパスして React のコードが正しいのに、実際のブラウザでバグが引き続き見える場合——ブラウザを確認してください。コンポーネントをフルページリロードなしに再マウントする SPA では、Chrome の認証情報マネージャーが `onChange` 経由で制御された入力に値を注入することがあります。修正はアプリケーションの状態変更ではなく、適切な `autoComplete` セマンティクスです。

---

## 2. GitHub リポジトリ移行

リポジトリが個人アカウント `Kazuma-Ishimine/TDD_todo_app` から organization `kishimin-ai-create/TDD_todo_app` に移管されました。ローカルのリモートをそれに合わせて更新しました：

```bash
git remote set-url origin git@github.com:kishimin-ai-create/TDD_todo_app.git
```

**CI（GitHub Actions）への影響**：なし。ワークフローファイルはリポジトリ内に存在し、リポジトリとともに移動します。Actions は再設定なしに継続して実行されます。

**Render デプロイへの影響**：場合によっては重大。Render の GitHub App は organization レベルで認可されています。リポジトリが個人アカウントから org に（またはその逆に）移動すると、Render は自動デプロイをトリガーするためのウェブフックアクセスを失う可能性があります。移行後に自動デプロイが停止した場合、GitHub の設定で新しい org に対して GitHub App の再認可が必要です。

---

## 3. Render デプロイのトラブルシューティング

Render のデプロイログに、正常な起動の前に `yarn start` の失敗が表示されていました。`render.yaml` でスタートコマンドが明示的に定義されているため、これは混乱を招きました：

```yaml
# render.yaml
services:
  - type: web
    name: tdd-todo-app-backend
    env: node
    buildCommand: cd backend && npm ci --include=dev && npm run build && node dist/infrastructure/migrate.mjs
    startCommand: cd backend && npm start   # ← 使用されるべきもの
```

`yarn start` の失敗は、Start Command フィールドが `render.yaml` に合わせて更新されていなかった **Render ダッシュボードの古いまたは手動設定のサービス** から来ていました。サービスが元々手動で作成された場合、Render では `render.yaml` の仕様とダッシュボードの設定が乖離することがあります。

**バックエンドはすでにポート10000で正常に起動していました** ——`yarn start` の失敗はプリフライトチェックまたは古い設定が最初に試みられたものであり、実際のサービス起動ではありませんでした。

修正方法：Render ダッシュボード → バックエンドサービス → Settings → Start Command を開き、`render.yaml` に合わせて `cd backend && npm start` と設定されていることを確認します。合わせると、エラーはログから消えます。

---

## まとめ

| トピック | 根本原因 | 修正 |
|---|---|---|
| Sign Up フォームのオートフィル | Chrome の認証情報マネージャーが SPA での React 再マウント時に制御された入力に `onChange` を発火させる | 正しい `autoComplete` 属性を追加：signup では `off`/`new-password`、login では `username`/`current-password` |
| リポジトリ移行 | 個人アカウント → org の GitHub 移管 | ローカルリモートURLを更新；新しい org に対して Render の GitHub App を再認可 |
| Render の `yarn start` エラー | ダッシュボードの Start Command が `render.yaml` と同期していない | ダッシュボードの設定を `cd backend && npm start` に合わせる |

今日の最も再利用可能な教訓：**ブラウザの動作とアプリケーションの状態は別のレイヤーです**。コードとテストが正しいのにブラウザでバグが続く場合、次のデバッグ対象はブラウザ自体です——そして `autoComplete` は、ブラウザがフォームにどれほど積極的に介入するかを制御するレバーの1つです。

1. **ナビゲーション全体で値を保持するサインアップ フォームに関するバグ レポート** — これはアプリケーションの状態ではなく、Chrome の自動入力が原因であることが判明しました
2. **GitHub リポジトリの移行** — 個人アカウントから組織アカウントへの移行。CI とデプロイメントに影響を及ぼします。
3. **レンダリング デプロイメントの混乱** - バックエンドが実際には正しく起動していたため、ログ内の `yarn start` エラーが誤解を招くものでした。

---

## 1. 存在しなかったバグ (Chrome が実現するまで)

### レポート

> 「サインアップ フォームに値を入力すると、[ログイン] に移動して、[サインアップ] に戻ります。以前の値がフィールドにまだ残っています。」

古典的なフォームの状態リーク。おそらく、グローバル スコープ内に存在する `useState` か、条件付きレンダリングされたコンポーネントで `key` プロパティが欠落している可能性があります。調査する時間です。

### 調査: すべての兆候は「バグなし」を示しています

フォームの状態は完全に `useAuthForm.ts` 内に存在します。

```ts
// frontend/src/features/auth/hooks/useAuthForm.ts
export function useAuthForm({ endpoint, fallbackErrorMessage, onSuccess }) {
  const [email, setEmail]       = useState('')
  const [password, setPassword] = useState('')
  // ...
}
```

ローカル `useState` — Jotai アトムなし、グローバル スコープなし。この状態は、コンポーネントがアンマウントされると消滅します。

次に、`App.tsx` がチェックされました。

```tsx
// frontend/src/App.tsx
if (!auth) {
  if (currentPage.name === 'signup') return <SignupPage key="signup" />
  if (currentPage.name === 'login')  return <LoginPage  key="login"  />
  return <LandingPage />
}
```

すべてのナビゲーションでクリーンなマウントを保証する 2 つのこと:

- `SignupPage` と `LoginPage` は **異なるコンポーネント タイプ** — React は、タイプが変更されるたびに古いインスタンスをアンマウントし、新しいインスタンスをマウントします
- どちらも明示的な `key` プロパティ (`key="signup"`、`key="login"`) を持っています。これは、2 つのページが単一の共有コンポーネントにリファクタリングされた場合でも状態の独立性を確保するために、特に前のコミット (`ca13d16`) で追加された防御手段です。

180 個のテストをすべて実行すると、正確な `signup → login → signup` ナビゲーション フローを実行し、各ステップでフィールドが空であることをアサートする `App.test.tsx` の回帰テストを含め、すべてが合格することが確認されました。

**コード レビューからの結論: アプリケーションの状態は正しく動作しています。**

### 本当の根本原因: Chrome の認証情報マネージャー

実際の原因は **React の再マウント時に起動されるブラウザの自動入力**でした。

この SPA のページ間を移動するときに Chrome で何が起こるかは次のとおりです。

1. ユーザーがサインアップページで電子メールとパスワードを入力する
2. Chrome の資格情報マネージャーは、新しい `<input type="email">` 要素と `<input type="password">` 要素を登録します
3. ユーザーがログインに移動します — React が `SignupPage` をアンマウントし、`LoginPage` をマウントします
4. ユーザーはサインアップに戻ります — React は `LoginPage` をアンマウントし、`SignupPage` をマウントします
5. **Chrome では、ページをリロードしなくても、新しい `<input>` 要素が同じ URL に表示されます**
6. Chrome は、制御された入力で `onChange` イベントをトリガーし、以前に入力した (または以前に保存した) 値を挿入します。
7. React はこれらの合成イベントを処理し、それに応じて `useState` を更新します。

入力は制御されているため (`value={email}` + `onChange`)、Chrome の自動入力は、DOM を直接変更するのではなく、`onChange` をディスパッチすることによって機能します。これにより、React の観点からは実際のユーザー入力と区別できなくなります。 `useState` 値はマウント時に完全にゼロでした。マウント直後に自動入力で書き込まれていました。

これは既知の SPA 固有の動作です。従来のページ ナビゲーションではブラウザの状態がフラッシュされますが、同じ URL での React の再マウントではフラッシュされません。

### 修正: `autoComplete` 属性

修正は小規模で、フォームごとに属性が 2 つだけですが、値の選択が重要です。

**`SignupPage.tsx`**

```tsx
<input
  id="email"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  autoComplete="off"          // ← added
/>
<input
  id="password"
  type="password"
  value={password}
  onChange={(e) => setPassword(e.target.value)}
  autoComplete="new-password" // ← added
/>
```

**`LoginPage.tsx`**

```tsx
<input
  id="email"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  autoComplete="username"         // ← added
/>
<input
  id="password"
  type="password"
  value={password}
  onChange={(e) => setPassword(e.target.value)}
  autoComplete="current-password" // ← added
/>
```

### なぜこれらの特定の値なのか?

|フィールド |値 |理由 |
|---|---|---|
|サインアップページの電子メール | `"off"` |新しいアカウント フォームでの自動入力を抑制します。ここには保存された資格情報を事前入力する必要はありません。
|サインアップページのパスワード | `"new-password"` |これがアカウント作成フォームであることを Chrome に通知します。 Chrome は新しい認証情報の保存を提案しますが、**以前に保存された認証情報は挿入されません**。
|ログインページの電子メール | `"username"` |ログイン電子メールの正しいセマンティック値。保存された資格情報と一致するようにブラウザのパスワード マネージャーに指示します。
|ログインページのパスワード | `"current-password"` | `username` と連携して、ブラウザのパスワード マネージャーがログイン ページに自動入力できるようにします。これが意図された UX |

重要な洞察: `new-password` と `current-password` は、Chrome が「アカウントの作成」と「サインイン」を区別するために使用するセマンティック シグナルです。これを正しく設定すると、ログイン時の意図した自動入力動作を維持しながら、サインアップ時の自動入力の問題が解決されます。

### 結果

- 修正後は 28/28 認証関連のテストに合格します
- 全体として 180 のテストすべてに合格
- テストの変更は必要ありませんでした。テスト環境 (Vitest + jsdom) はブラウザーの自動入力をエミュレートしないため、既存の回帰テストはすでに基礎となる React 状態の動作を正しくカバーしていました。

### レッスン

180 個の単体テスト/統合テストがすべて合格し、React コードが正しいにもかかわらず、実際のブラウザーにバグがまだ表示されている場合は、ブラウザーを確認してください。ページ全体をリロードせずにコンポーネントを再マウントする SPA では、Chrome の資格情報マネージャーは、`onChange` を介して制御された入力に値を挿入できます。修正は、アプリケーションの状態の変更ではなく、適切な `autoComplete` セマンティクスです。

---

## 2. GitHub リポジトリの移行

リポジトリは、個人アカウント `Kazuma-Ishimine/TDD_todo_app` から組織 `kishimin-ai-create/TDD_todo_app` に転送されました。ローカル リモートは以下に一致するように更新されました。

```bash
git remote set-url origin git@github.com:kishimin-ai-create/TDD_todo_app.git
```

**CI への影響 (GitHub アクション)**: なし。ワークフロー ファイルはリポジトリ自体の内部に存在し、リポジトリとともに移動します。アクションは再構成せずに実行を継続します。

**レンダリングの展開への影響**: 重大な影響を与える可能性があります。 Render の GitHub アプリは組織レベルで承認されています。リポジトリが個人アカウントから組織に移動すると (またはその逆)、Render は自動デプロイをトリガーするための Webhook アクセスを失う可能性があります。 GitHub アプリは、GitHub の設定で新しい組織に対して再認証する必要があります。移行後に自動デプロイが起動しなくなった場合は、最初に確認する場所です。

---

## 3. レンダリングのデプロイメントのトラブルシューティング

Render のデプロイ ログには、起動が成功する前に `yarn start` の失敗が示されていました。 `render.yaml` は開始コマンドを明示的に定義しているため、これは混乱を招きました。

```yaml
# render.yaml
services:
  - type: web
    name: tdd-todo-app-backend
    env: node
    buildCommand: cd backend && npm ci --include=dev && npm run build && node dist/infrastructure/migrate.mjs
    startCommand: cd backend && npm start   # ← what should be used
```

`yarn start` エラーは、**レンダリング ダッシュボードの古いサービスまたは手動で構成されたサービス**に起因しており、開始コマンド フィールドが `render.yaml` と一致するように更新されていませんでした。サービスが最初に手動で作成された場合、レンダリングでは、`render.yaml` 仕様とサービスのダッシュボード設定に保存された内容との間に切断が発生する可能性があります。

**バックエンドはすでにポート 10000 で正常に実行されていました** — `yarn start` の失敗は、実際のサービスの開始ではなく、プリフライト チェックまたは最初に試行された古い構成によるものでした。

修正: [レンダリング ダッシュボード] → [バックエンド サービス] → [設定] → [コマンドの開始] に移動し、`render.yaml` と一致する `cd backend && npm start` と表示されていることを確認します。調整後、エラーはログから消えます。

---

## まとめ

|トピック |根本原因 |修正 |
|---|---|---|
|サインアップフォームの自動入力 | Chrome の認証情報マネージャーは、React が制御された入力を SPA に再マウントするときに、制御された入力に対して `onChange` をディスパッチします。正しい `autoComplete` 属性を追加します: サインアップ時は `off`/`new-password`、ログイン時は `username`/`current-password` |
|リポジトリの移行 |個人→組織 GitHub 転送 |ローカルのリモート URL を更新します。新しい組織用に Render の GitHub アプリを再認証する |
| `yarn start` エラーをレンダリングします。ダッシュボード開始コマンドが `render.yaml` と同期していません |ダッシュボード設定を `cd backend && npm start` に合わせる |

今日の最も再利用可能なポイント: **ブラウザの動作とアプリケーションの状態は別のレイヤーである**。コードとテストは正しいが、ブラウザーにバグが残っている場合、ブラウザー自体が次のデバッグ ターゲットになります。`autoComplete` は、ブラウザーがフォームとどの程度積極的に対話するかを制御するレバーの 1 つです。
