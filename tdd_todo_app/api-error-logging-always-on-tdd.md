# API エラー ロギングの常時オン: 完全な TDD サイクル

## 対象読者

小規模ながら具体的なエンドツーエンドの TDD サイクルを確認したいバックエンド エンジニア
本番環境に不可欠な機能 — バグの発見、失敗するテストの作成、適用まで
最小限の修正、リファクタリング、コードレビューへの対応。

---

## 背景: エラーを黙らせたバグ

`backend/src/infrastructure/hono-app.ts` ファイルにはすでに機能する API ログが含まれていました
このサイクルの前のミドルウェア。次の 2 つの関数が出力を処理しました。

- `logSuccessRequest()` — 2xx 応答の `[METHOD] path → status (Xms)` を出力
- `logErrorRequest()` — 4xx/5xx 応答の `[METHOD] path → ERROR status — code: message` を出力

両方の関数は同じガードで開始されました。

```typescript
if (process.env.LOG_API_REQUESTS !== 'true') {
  return;
}
```

`logSuccessRequest()` にとってガードは理にかなっています。トラフィックの多い本番環境の展開
200 件ごとにログを記録する必要はありません。オペレーターは `LOG_API_REQUESTS=true` でオプトインします。

しかし、`logErrorRequest()` には **まったく同じガード**がありました。結果: どのような場合でも
`LOG_API_REQUESTS` が存在しなかった環境 (本番環境のデフォルト、CI、最もローカルな環境)
実行されます）、**すべてのエラー応答は黙って飲み込まれました**。 401、409、500 — 何もありません
ログに現れました。 `docs/spec/features/api-logging.md` の機能仕様は次のとおりです。
明示的:

> エラー応答 (4xx、5xx) は、
> `LOG_API_REQUESTS` 環境変数。これにより、エラーが黙って発生することがなくなります。
> あらゆる環境に飲み込まれます。

実装が仕様と一致しませんでした。修正されたのは 1 つの削除されたブロックでした。 TDD
その理由を正確に文書化したサイクル。

---

## レッドフェーズ: 契約を明らかにするテストの作成

新しい `describe` ブロックが追加されました
`backend/src/tests/integrations/infrastructure/hono-app.medium.test.ts`:

```typescript
describe('Error logging always-on behavior (no LOG_API_REQUESTS env var)', () => {
  let consoleLogSpy: ReturnType<typeof vi.spyOn>;

  beforeEach(() => {
    // Guarantee the var is absent — this is the exact production default
    delete process.env.LOG_API_REQUESTS;
    consoleLogSpy = vi.spyOn(console, 'log').mockImplementation(() => {});
  });

  afterEach(() => {
    consoleLogSpy.mockRestore();
    delete process.env.LOG_API_REQUESTS;
  });
  // ...
});
```

`beforeEach` は、`delete process.env.LOG_API_REQUESTS` を設定するのではなく呼び出します。
`undefined` または `'false'` まで。これは生産を正確に反映しています。変数は次のとおりです。
存在しない、値が設定されていない。このブロック内のすべてのテストは、環境変数なしで実行されます。

5 つのテスト ケースが作成されました。

| # |テスト |検証する内容 |
|---|------|-----------------|
| 1 | `should log 4xx error response even when LOG_API_REQUESTS is not set` |空の本体を持つ `POST /api/v1/apps` は、422 **および** 一致するログ エントリを生成します。
| 2 | `should log 401 error response even when LOG_API_REQUESTS is not set` |存在しない資格情報によるログインは `[POST] … → ERROR 401` ログに記録されます。
| 3 | `should log 409 conflict error even when LOG_API_REQUESTS is not set` |サインアップ ログ `→ ERROR 409` が重複し、`EMAIL_ALREADY_EXISTS` が含まれています。
| 4 | `should NOT log successful 2xx response when LOG_API_REQUESTS is not set` | `GET /api/v1/apps → 200` は **ログ エントリを生成しません** |
| 5 | `should log 4xx error but NOT log 2xx success when LOG_API_REQUESTS is not set` |混合シナリオ: 422 はログに記録され、200 はログに記録されません。

テスト 3 は、サインアップという 2 段階のセットアップが必要なため、詳しく調べる価値があります。
一度アカウントを作成してから、重複してサインアップを試みます。

```typescript
it('should log 409 conflict error even when LOG_API_REQUESTS is not set', async () => {
  const { app } = buildApp();

  // First signup (201) — success logs are suppressed without LOG_API_REQUESTS
  await req(app, 'POST', '/api/v1/auth/signup', {
    email: 'always-on@example.com',
    password: 'password123',
  });
  // 201 success is not logged without LOG_API_REQUESTS — no mockClear() needed here

  // Duplicate signup → 409
  const res = await req(app, 'POST', '/api/v1/auth/signup', {
    email: 'always-on@example.com',
    password: 'password123',
  });
  expect(res.status).toBe(409);

  // Assert both the presence and the format
  const errorLogCall = consoleLogSpy.mock.calls.find((call: unknown[]) =>
    typeof call[0] === 'string' &&
    call[0].includes('ERROR') &&
    call[0].includes('409')
  );
  expect(errorLogCall).toBeDefined();
  const logMessage = errorLogCall![0] as string;
  expect(logMessage).toMatch(/→ ERROR 409/);
  expect(logMessage).toContain('EMAIL_ALREADY_EXISTS');
});
```

修正前は、このブロック内の 5 つのテストはすべて失敗していました: `consoleLogSpy`
`logErrorRequest()` がガードですぐに戻ったため、ゼロ コールをキャプチャします。

---

## グリーンフェーズ: たった 1 行の修正

失敗したテストがコミットされたため、修正は簡単でした。ガードブロックは、
`logErrorRequest()` から削除されました:

**前に：**
```typescript
async function logErrorRequest(
  method: string, path: string, status: number, context: Context
): Promise<void> {
  if (process.env.LOG_API_REQUESTS !== 'true') {  // ← guarded
    return;
  }
  const errorDetails = await extractErrorDetails(context);
  // ...
}
```

**後：**
```typescript
async function logErrorRequest(
  method: string, path: string, status: number, context: Context
): Promise<void> {
  // No guard — always logs regardless of LOG_API_REQUESTS
  const errorDetails = await extractErrorDetails(context);
  // ...
}
```

JSDoc は、新しい契約を反映するために同じコミットで更新されました。

```typescript
/**
 * Logs an error API request with format: [METHOD] path → ERROR status — code: message
 * If error details cannot be extracted, falls back to: [METHOD] path → ERROR status
 * Always logs regardless of LOG_API_REQUESTS environment variable.
 */
```

`logSuccessRequest()` はそのまま残されました。そのガードは意図的であり、正しいものです。

仕様に合わせた結果の動作は次のようになります。

|応答タイプ |ステータス範囲 | `LOG_API_REQUESTS=true` | `LOG_API_REQUESTS` がありません |
|---|---|---|---|
|成功 | 2xx |記録済み | **サイレント** |
|エラー | 4xx、5xx |記録済み | **ログに記録されました** |

削除後、342 の統合テストはすべて合格しました。

---

## リファクタリング フェーズ: 次に進む前に明確にする

緑になった後、2 回の独立したクリーンアップが続きました。

### `extractErrorDetails()` のガード句

この関数は以前、応答本文を検証するためにネストされた if/else ブロックを使用していました。
構造：

```typescript
// Before — nested
if (typeof responseBody === 'object' && responseBody !== null && 'error' in responseBody) {
  const errorObj = responseBody.error;
  if (typeof errorObj === 'object' && /* ... more checks ... */) {
    return { code: errorObj.code, message: errorObj.message };
  } else {
    console.warn('[logging] Error object has unexpected structure …');
  }
} else {
  console.warn('[logging] Response body missing "error" property');
}
```

リファクタリングにより、各条件がガード句 (失敗時の早期復帰) に反転されました。
ド・モルガンの法則を適用して小切手を反転します。

```typescript
// After — guard clauses
if (typeof responseBody !== 'object' || responseBody === null || !('error' in responseBody)) {
  console.warn('[logging] Response body missing "error" property');
  return null;
}

const errorObj = responseBody.error;

if (
  typeof errorObj !== 'object' ||
  errorObj === null ||
  !('code' in errorObj) ||
  !('message' in errorObj) ||
  typeof errorObj.code !== 'string' ||
  typeof errorObj.message !== 'string'
) {
  console.warn('[logging] Error object has unexpected structure or non-string code/message');
  return null;
}

return { code: errorObj.code, message: errorObj.message };
```

ハッピー パス `return` は現在最下位にあり、すべてのガードが完了した後にのみ到達します。
合格した。ネスト レベルが 1 レベル下がり、ロジック フローが 1 つになりました。
方向。動作は同じです。変更は純粋に構造的なものです。

### 変数の名前変更: `elapsedTime` → `elapsedTimeMs`

ミドルウェアの `afterEach` ハンドラー内で、経過時間変数の名前が変更されました。

```typescript
// Before
const elapsedTime = Date.now() - startTime;
logSuccessRequest(method, path, status, elapsedTime);

// After
const elapsedTimeMs = Date.now() - startTime;
logSuccessRequest(method, path, status, elapsedTimeMs);
```

接尾辞 `Ms` は、宣言サイトで単位を明示的にします。これは、次の場合に重要になります。
ログ出力 `(${elapsedTimeMs}ms)` の読み取り — ユニットが 2 回表示され、それらは
同意する。小さな変更ですが、次のエンジニアが本書を読む際の曖昧さが解消されます。
コード。

---

## レビュー段階: 優先度の低い 3 つの調査結果が修正されました

新しいテスト ブロック (`review/api-logging-20250725.md`) のコード レビューでは、次の 3 つの問題が発生しました。
P3 の所見。すべては 1 回のフォローアップ コミットで修正されました。

### 調査結果 1 — テスト 1 では、フォーマットではなく存在のみをチェックしました

422 常時オン テストの元のアサーションでは、文字列 `'ERROR'` が検索されました。
および `'422'` はログ メッセージ内の任意の場所にあります。次のような仮想の文字列
`"ERROR: status 422 happened"` は、そうでない場合でもそのチェックを満たします。
仕様形式 `[METHOD] path → ERROR status — code: message` と一致します。

**修正:** 方法と一致して、存在チェックの後に正規表現アサーションを追加しました。
テスト 2 ～ 5 はすでに書かれています。

```typescript
expect(errorLogCall).toBeDefined();
const logMessage = errorLogCall![0] as string;
expect(logMessage).toMatch(/\[POST\].*→ ERROR 422/);  // ← added
```

### 調査結果 2 — 新しいブロックで `console.warn` が抑制されていない

`extractErrorDetails()` は、抽出失敗パスごとに `console.warn()` を呼び出します。
(不正な JSON、`error` プロパティの欠落、間違った構造、サイズ ガードの超過)。
既存の `beforeEach` は `console.log` のみをモックしており、警告呼び出しは無料のままです。
CI 出力に漏れます。

**修正:** `consoleLogSpy` と一緒に `consoleWarnSpy` を追加しました:

```typescript
let consoleWarnSpy: ReturnType<typeof vi.spyOn>;

beforeEach(() => {
  delete process.env.LOG_API_REQUESTS;
  consoleLogSpy = vi.spyOn(console, 'log').mockImplementation(() => {});
  consoleWarnSpy = vi.spyOn(console, 'warn').mockImplementation(() => {});
});

afterEach(() => {
  consoleLogSpy.mockRestore();
  consoleWarnSpy.mockRestore();
  delete process.env.LOG_API_REQUESTS;
});
```

### 調査結果 3 — 409 テストの `mockClear()` はサイレント no-op でした

409 テストは、重複 409 をトリガーする前に 1 回 (201) サインアップしました。
最初のサインアップでは、`consoleLogSpy.mockClear()` という元のコードが使用されます。しかし、なぜなら
`LOG_API_REQUESTS` はこのブロック全体に存在せず、`logSuccessRequest()` が返されます。
早めに `console.log` を呼び出すことはありません。つまり、スパイはその時点で呼び出しを保留します。
ポイントし、`mockClear()` は何もクリアしません。

**修正:** 呼び出しが削除され、不要な理由を説明するコメントに置き換えられました。

```typescript
// 201 success is not logged without LOG_API_REQUESTS — no mockClear() needed here
```

これにより、スパイではないかと疑問に思う将来の読者に対して、その意図が明確になります。
クリア前に意味深な何かを捉えていました。

ファイル内の 56 個のテストはすべて、これらの各修正後も引き続き合格しました。

---

## このサイクルから得た教訓

**1.対称ガードは常に正しいとは限りません。**
ガードを 1 つの関数から兄弟関数にコピーするのは、見栄えがするため魅力的です。
一貫性のある。しかし、`logSuccessRequest()` と `logErrorRequest()` は基本的に
さまざまなオブザーバビリティ契約。成功ログはオプションのノイズです。エラーログは
必須の信号。対称性は一貫性として隠れていたバグでした。

**2.環境変数を削除する `beforeEach` は、環境変数を偽の値に設定するものよりも強力です。**
`delete process.env.LOG_API_REQUESTS` は正確にミラーリングします。「これではデプロイされていません」
変数」。`process.env.LOG_API_REQUESTS = undefined` または `= 'false'` は、 —
`'false' !== 'true'` は、早期リターン チェックに対して `true` を返しますが、
意図が濁っている。 「変数が存在しない」セマンティクスには `delete` を使用します。

**3.ガード句は、複雑な形状を検証する際の認知的負荷を軽減します。**
`extractErrorDetails()` リファクタリングは教科書的な例です。各ガードは以下を排除します。
状態が悪く、すぐに戻ります。実行が底に達するまでに、すべてが
不変条件が証明される。幸せな道は直線のように見えます。

**4.空のスパイの `mockClear()` は無害であるだけでなく、誤解を招きます。**
スパイが排除され、読者が通話をキャプチャしたかどうかを判断できない場合、彼らは
明確かどうかを理解するには、以前の実行全体を推論する必要があります。
重要だった。 1 行のコメントがその理由を示します。

**5.存在チェックと形式チェックは異なる目的を果たします。**
レビュー修正後のテスト 1 には 2 つのアサーションがあります: `expect(errorLogCall).toBeDefined()`
ログがまったく出力されたことを確認します。 `expect(logMessage).toMatch(/\[POST\].*→ ERROR 422/)`
仕様の形式と一致することを確認します。存在を確認するだけのテストは、長期間にわたって存続します。
形式を変更する回帰。両方とも保管してください。

---

## まとめ

|フェーズ |コミット |変更 |
|-------|--------|--------|
|赤 | `ed5da5a` | 5 つの新しいテスト: `describe('Error logging always-on behavior …')` |
|緑 | `ed5da5a` | `logErrorRequest()` から `LOG_API_REQUESTS` ガードを削除 |
|リファクタリング | `3f13b58` | `extractErrorDetails()` のネストされた if/else をフラット化して句を保護する |
|リファクタリング | `103a59c` | `elapsedTime` → `elapsedTimeMs` に名前変更 |
|ドキュメントを確認する | `be0b542` |レビュー結果ドキュメントを追加しました (`review/api-logging-20250725.md`) |
|修正をレビューする | `a59aea0` |テスト 1 でアサーションをフォーマットします。 `consoleWarnSpy`;削除された no-op `mockClear()` |

根本的なバグは単一のガードが誤って適用されたことでした。それは、決して適用されるべきではない 1 つの `if` ブロックです。
`logErrorRequest()`にありました。 TDD サイクルにより、そのギャップが 5 つの実行可能ファイルに変わりました。
テストを行って、1 回の削除で修正し、周囲のコードの読みやすさを改善しました。
レビューを通じて 3 つの微妙なテスト品質の問題を発見 - コードベースを残す
最初よりも明らかになりました。
