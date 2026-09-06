# VercelがReadyでも404になるときはFramework Presetを確認する

## 結論

VercelのDeploymentが`Ready`でも、配信URLがVercel基盤の`404 NOT_FOUND`になる場合がある。今回のモノレポではRoot Directoryは正しく`frontend`を指していたが、Framework Presetが`Other`になっていた。

Framework Presetを`Next.js`へ変更して再デプロイした結果、固定Production Domainの`/`と、Next.jsがCloudflare Workersへ中継する`/api/diaries`がどちらも200を返した。

## 発生した問題

### 症状

Vercel Dashboardでは複数のProduction Deploymentが`Ready`と表示されていた。しかし、Production Domainと生成Deployment URLへアクセスすると次の応答になった。

```text
HTTP/1.1 404 Not Found
X-Vercel-Error: NOT_FOUND
Content-Type: text/plain; charset=utf-8
```

レスポンス本文はNext.jsの404ページではなく、Vercelが返す短い`The page could not be found`だった。

### 発生条件

- Repository: backendとfrontendを分けたモノレポ
- Frontend: Next.js 16.2.6
- Package manager: Bun、`frontend/bun.lock`
- Root Directory: `frontend`
- Framework Preset: `Other`

### 影響

Cloudflare Workersのbackendは正常に動作していたが、利用者がVercelのFrontendへアクセスできず、Frontend proxy経由のAPIも確認できなかった。

## 調査

### Deployment Protectionを確認した

最初の生成Deployment URLはVercel Authenticationへ302 redirectしたため、Deployment Protectionを確認した。その後は認証redirectではなく`X-Vercel-Error: NOT_FOUND`を伴う404へ変わった。

認証問題と404は別の状態だった。後者はアプリケーションが返した404ではなく、Vercelが配信対象を見つけられない応答だった。

### Next.jsのルートを確認した

リポジトリには`frontend/app/page.tsx`が存在した。`/`の実装がないという仮説とは一致しなかった。

### Root DirectoryとFramework Presetを確認した

Vercel Project SettingsではRoot Directoryが`frontend`に設定されていた。一方、Framework Presetは`Other`で、画面には汎用的なBuild CommandとOutput Directoryの既定値が表示されていた。

`frontend/package.json`には次の実在するbuild scriptがある。

```json
{
  "scripts": {
    "build": "next build"
  }
}
```

さらに`frontend/bun.lock`とNext.js依存が存在するため、Framework Presetを`Next.js`へ変更し、Vercelの自動設定を利用した。

## 原因

今回の404は、Root DirectoryではなくFramework Presetが`Other`だった状態と結び付いていた。Deploymentは`Ready`になっていたが、Next.jsアプリケーションとして期待する配信結果になっていなかった。

Framework Preset変更後に正常化したことは確認できた。ただし、Vercel内部でどの出力認識が失敗していたかまではログで確認していないため、内部原因は未確認である。

## 解決方法

Vercel Project SettingsのBuild and Deploymentで次の値を設定した。

| 項目 | 設定 |
| --- | --- |
| Framework Preset | `Next.js` |
| Root Directory | `frontend` |
| Build Command | Overrideしない |
| Output Directory | Overrideしない |
| Install Command | Overrideしない |

設定後、Production Deploymentを再実行した。Next.jsのbuild、`.next`出力、Bunによる依存関係のinstallはFramework Presetの自動設定へ委ねた。

## 動作確認

固定Production Domainへブラウザと同じGETを実行した。

```text
GET /             -> 200 text/html
GET /api/diaries -> 200 application/json
```

`/api/diaries`の成功により、VercelのNext.js Route Handlerから`BACKEND_URL`に設定したCloudflare Workerまでの経路も確認できた。

以前使っていた生成Deployment URLの1つは、修正後も`X-Vercel-Error: NOT_FOUND`を返した。検証と公開には、現在のDeploymentへ割り当てられた固定Production Domainを使用した。

## 再発防止

- モノレポではRoot DirectoryとFramework Presetを別々に確認する
- `Ready`表示だけで公開成功とせず、固定Production DomainへGETする
- HTMLだけでなく、server-side proxyを通るAPI endpointも確認する
- `X-Vercel-Error`の有無でVercel基盤の404とNext.jsの404を区別する
- Build CommandとOutput Directoryは、必要な理由がない限りNext.js Presetの自動設定を使う

## まとめ

- 症状: Vercelで`Ready`のDeploymentが`404 NOT_FOUND`を返した
- 確認事項: Root Directoryは正しかったがFramework Presetは`Other`だった
- 解決: Framework Presetを`Next.js`へ変更して再デプロイした
- 結果: 固定Production Domainの画面とFrontend proxy APIが200を返した
- 制約: Vercel内部で失敗していた出力認識の詳細は未確認である

## 参考資料

- [VercelでDeployment成功後の404を調査する](https://vercel.com/kb/guide/why-is-my-deployed-project-giving-404)
- [Vercel Project Settings](https://vercel.com/docs/project-configuration/project-settings)
- 確認日: 2026年9月6日
