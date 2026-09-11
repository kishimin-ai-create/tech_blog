# 地図と広告の障害を主要UIから分離する状態設計

## 結論

ランダム郵便番号アプリの主要状態は、郵便番号APIの`idle`、`loading`、`success`、`error`として管理し、地図、広告、Consentは別の任意機能として扱う。

```ts
type GeneratorState =
  | { status: "idle" }
  | { status: "loading"; previousResult?: PostalCode }
  | { status: "success"; result: PostalCode }
  | { status: "error"; error: UiError; previousResult?: PostalCode };
```

地図や広告の失敗をGenerator全体の`error`へ昇格させないことで、取得済みの郵便番号、住所、履歴、再生成を利用可能なまま保てる。

## 背景

ZipnamiのWeb UIはGoogle Maps Embed APIとGoogle AdSense、Android UIは外部地図アプリとAdMob・UMPを利用する。一方、製品の中心機能は郵便番号と住所の生成・表示である。

これらを1つのpage-level loading/errorへまとめると、任意サービスの障害が主要機能まで停止させる。成功済み結果を再取得中に消す設計も、network障害時に利用者が直前の情報を参照できなくする。

## 解決したい課題

- 排他的な主要状態を型で表現する。
- 再生成中と失敗後も直前結果を保持する。
- 成功したrequestだけを履歴へ1回追加する。
- Maps、広告、Consent、外部アプリの部分障害を局所化する。
- WebとAndroidで同じdomain behaviorを維持する。

## 前提・制約

- 初期表示でAPIを自動呼出ししない。
- Loading中は重複生成だけを無効にし、navigationと履歴を止めない。
- 履歴はclient内だけに最大20件保存し、重複を保持する。
- Androidへ埋込Google Maps SDKや位置情報権限を追加しない。
- 広告とConsentの失敗は主要操作を妨げない。

## 検討した選択肢

### Option A: Page全体を1つのloading/errorで管理する

状態数は少なく見えるが、地図や広告の失敗理由まで主要API失敗と同じ扱いになる。再生成時にpage全体をspinnerへ置き換えると、直前結果や履歴も操作できない。

### Option B: Generatorと任意integrationの状態を分ける

GeneratorはAPI結果だけを所有し、Maps、広告、Consentはそれぞれ局所的なloading・unavailable状態を持つ。GeneratorのLoading・Errorは`previousResult`を保持できる。

## 評価軸

| 評価軸 | Page全体で管理 | 境界ごとに管理 |
| --- | --- | --- |
| 部分障害 | 主要機能まで停止しやすい | 失敗した領域へ限定できる |
| 直前結果 | 消えやすい | 保持を型で表現できる |
| 不正状態 | 複数booleanで生じやすい | Discriminated Unionで排除できる |
| 実装量 | 初期は少ない | 境界ごとの状態が必要 |
| テスト | Page全体の組合せが増える | Feature単位で検証できる |

## 最終判断

Generatorの排他的状態をDiscriminated Unionで表現し、Maps、広告、Consent、外部地図起動を独立したadapter・表示境界へ分ける設計を採用した。

成功時だけ現在結果を置換して履歴へ追加する。失敗時は`previousResult`を表示可能に保ち、同じ結果を履歴へ再追加しない。地図失敗時は選択住所と外部linkを、外部地図起動失敗時は現在結果を維持する。

## なぜこの選択をしたか

主要機能と任意integrationは、利用者価値と障害原因が異なる。状態所有者を分けることで、どの失敗がどの表示を無効にするかを明示できる。

また、WebとAndroidでcomponent自体を共有しなくても、Generatorの状態遷移と履歴不変条件を共有契約として揃えられる。Platform固有の地図・広告処理はadapter境界の内側へ閉じ込められる。

## 現在の結果

この判断は英語版・日本語版のZipnami UI設計書へ反映した。Webの情報順、Androidのbanner領域、Accessibility、Responsive検証、Small・Medium・Largeのtest boundaryまで文書化している。

2026年9月11日時点では設計書だけが完成しており、React・Expoのcomponent実装と自動テストは未着手である。実装後の操作性や障害時挙動は未検証である。

根拠となるコミット:

- `e6a4ce5 docs: define Zipnami API and UI contracts`
- `7cf01de docs: add Japanese API and UI designs`

## トレードオフ・今後の懸念

- 状態境界とadapterが増えるため、単一page stateよりファイル数が増える。
- SDK固有の広告未配信・Consent失敗状態は、導入するversionの契約確認が必要である。
- `previousResult`と履歴への追加を別々に扱い、retryで二重登録しないテストが必要である。
- 自動Accessibility検査だけではscreen readerやAndroid実機の操作を保証できないため、手動検証が残る。

## まとめ

外部サービスを使うUIでは、Pageに存在するすべての機能を同じ成功・失敗状態へ入れない。製品の中心となるGenerator状態と任意integrationの状態を分け、直前結果を保持することで、部分障害が主要価値を奪わない設計にできる。

確認日: 2026年9月11日
