# HTMLが異なる複数サイトを共通モデルへ収集するAdapter設計

## 結論

複数サイトの記事を同じDBへ保存するときは、サイト固有のURLとDOM解析をSource Adapterへ閉じ込め、Application Serviceには正規化済みの共通モデルだけを渡す。DBでは外部IDと収集元の組み合わせを一意にし、記事と画像を1トランザクションでupsertすると、再実行、途中中断、記事更新を同じ経路で扱える。

本稿は設計段階の記録であり、スクレイパー実装や本番収集の結果ではない。

## 背景

対象となった2サイトは、どちらも一覧ページから数値IDを含む詳細ページへ遷移し、詳細ページにタイトル、執筆者、公開日時、本文、画像を持っていた。一方、URL path、CSS class、本文のDOM境界は一致しなかった。

共通処理へselectorを直接書くと、1サイトのDOM変更が収集全体へ波及する。また、サイトごとに保存処理まで実装すると、重複判定やトランザクションの規則が分岐する。

## 解決したい課題

- サイト固有のHTML変更を局所化する
- 一覧と詳細から得た値を同じ検証規則へ通す
- 同じ記事を再取得しても重複させない
- 記事本文と画像の部分保存を防ぐ
- 中断後に専用ジョブテーブルなしで再開する

## 前提・制約

- HTTP取得とHTML解析は外部システムを扱うInfrastructureである。
- 収集順序と保存単位はApplication Serviceが所有する。
- Domain ModelはHTTPライブラリ、HTML parser、ORMへ依存しない。
- 本番サイトは通常の自動テストから呼び出さない。
- DOMから必須値を取得できない場合、ページ全体を本文にするfallbackは使用しない。

## 検討した選択肢

### Option A: 共通スクレイパーへサイト別条件を追加する

URLやselectorごとに条件分岐し、解析から保存までを1つの処理にまとめる。

#### メリット

- 最初のファイル数を抑えられる。
- 小規模な試作では処理を追いやすい。

#### デメリット

- HTML差分、再試行、保存処理の分岐が同じ場所へ集まる。
- サイト追加やDOM変更の影響範囲が広がる。
- parserだけを縮約fixtureで検証しにくい。

### Option B: Source Adapterで共通モデルへ変換する

各Adapterが一覧URL、詳細URL、selector、日時形式を担当し、共通の`CollectedRecord`を返す。保存処理は共通Repositoryへ集約する。

#### メリット

- サイト固有変更をAdapterとfixtureへ限定できる。
- Application ServiceとRepositoryを実サイトなしで検証できる。
- 保存、重複、更新、rollbackの規則を1つにできる。

#### デメリット

- Adapterごとの契約とテストが必要になる。
- 共通化できない差分を無理にDomainへ持ち込まない判断が必要になる。

## 評価軸

| 評価軸 | 共通処理内の条件分岐 | Source Adapter |
| --- | --- | --- |
| DOM変更の局所化 | 低い | 高い |
| parserの単体検証 | 分岐が増える | Adapter単位で可能 |
| 保存規則の一貫性 | 分岐しやすい | Repositoryへ集約 |
| 初期ファイル数 | 少ない | 多い |
| サイト追加 | 共通処理を変更 | Adapterを追加 |

## 最終判断

サイトごとにSource Adapterを置き、共通のPortを実装する。Application ServiceはPortだけを参照し、URLとselectorを知らない。

```text
CollectionService
  |-- SourceAdapter Port
  |     |-- SourceAAdapter
  |     `-- SourceBAdapter
  |
  `-- RecordRepository Port --> MySQL Repository
```

Adapterの出力は次の共通形に揃える。

```text
CollectedRecord
  source_key
  entity_external_key
  external_key
  title
  body_html
  source_path
  published_at
  assets[]
```

`external_key`には詳細URLの数値IDを使う。執筆者名は変更される可能性があるため、Entityの一意キーにはサイト側のメンバーIDを使う。

## DB保存を冪等にする

記事の一意性は`source_id + external_key`で表現する。同じ記事を何度取得しても、insertではなく既存行の確認またはupdateになる。

1記事の保存は次の単位で行う。

1. Entityを外部IDでupsertする。
2. Recordを一意キーで取得する。
3. 正規化後の本文、タイトル、日時、画像順からSHA-256を計算する。
4. hashが同じなら収集日時だけを更新する。
5. hashが異なるならRecordを更新し、Asset集合を置換する。
6. RecordとAssetの処理を同じトランザクションでcommitする。

これにより、中断後は同じコマンドを再実行できる。専用チェックポイントがなくても、一意制約が保存済み境界として働く。

## 差分収集で単一の既存IDを終了条件にしない

一覧の途中に固定表示や公開順の入れ替わりがある場合、最初の登録済みIDで停止すると、その後ろの新規記事を取りこぼす可能性がある。

設計ではページ単位で終了を判断する。

- ページ内の全IDが登録済みである
- 直前ページまでに未登録IDがなかった

この両方を満たしたときに日次収集を終了する。直近期間は登録済み記事も再取得し、content hashによって編集を検出する。

## DOM契約違反をデータ汚染へ変えない

必須selectorが見つからない場合は`ParseContractError`とする。サイト全体の`body`や近い親要素をfallbackとして保存すると、ナビゲーション、プロフィール、関連記事まで本文へ混入するためである。

自動テストでは、実記事をコピーせずに必要最小限へ縮約したHTML fixtureを用意する。

- 一覧と詳細の正常形
- 画像なし、複数画像
- 必須DOM欠落
- 日時形式不正
- ページ循環
- 同じIDの再取得と本文更新

## 実装後の結果

未実装である。確認済みなのは、2サイトの公開HTMLに一覧、詳細ID、タイトル、執筆者、日時、本文、画像、ページリンクに相当する構造が存在することと、設計書のMarkdown構造である。

実装後は、縮約fixtureによるparserテスト、MySQLを使うRepositoryテスト、コマンドからDBまでの収集テストで結果を確認する必要がある。

## トレードオフ・今後の懸念

- DOM classは外部契約ではないため、予告なく変わり得る。
- 記事削除、執筆者移動、公開日時修正の扱いをfixtureで固定する必要がある。
- 本番収集前に利用規約、robots.txt、保存・利用範囲を人間が確認する必要がある。
- HTMLサニタイズとresponse size上限は、実装時にセキュリティレビューが必要である。

## まとめ

複数サイトの収集では、HTMLの似ている部分を共通化するより、異なる部分をAdapter境界へ隔離する方が変更に耐えやすい。共通モデルの後段へ検証、content hash、一意制約、トランザクションを置くことで、収集とDB保存を一つの再実行可能な流れとして扱える。
