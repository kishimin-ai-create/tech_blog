# 公開カタログのテストでreflectionより直接assertionを選ぶ

「定義済みの値をすべて公開しているか」をテストするとき、reflectionやLINQでプロパティ一覧を走査したくなる。しかし、テスト側の処理が複雑になるほど、何を仕様として確認しているのか読み取りにくくなる。本記事では、Mojicaの`ModelValidationReason`テストを直接assertionへ整理した過程と、その適用範囲を説明する。

## 当初の狙いは閉じた生成経路の検査だった

テスト計画`ERROR-02`は、仕様外の検証理由を生成できないことを表そうとしていた。この契約を実行時に調べる案として、公開コンストラクターやfactory methodをreflectionで列挙するテストが検討された。

しかし、公開生成経路を漏れなく定義するには、次のような要素を扱う必要がある。

- public constructor
- static factory method
- instance methodが同じ型を返す経路
- implicit／explicit conversion operator
- recordが生成する特殊メンバー

`BindingFlags`や`IsSpecialName`の条件が少し違うだけで検査漏れが生じる。結果として、Domain契約よりreflection APIの正しさを検証する比重が大きくなった。

## テストの問いを分離する

そこで、1つのテストへ異なる問いを詰め込まず、次の2件へ分離した。

1. `ERROR-02`: 未定義理由を生成できないか。
2. `REASON-01`: 仕様書にある理由が期待する値を公開しているか。

`ERROR-02`は、適切な検証境界が決まるまでSkippedの計画として残した。一方、現在の公開値は通常のユニットテストで明確に確認できるため、`REASON-01`として実装した。

この分離により、未検証の契約を「実装済み」に見せず、確認できる事実だけをGreenにできる。

## 配列やLINQを使わず、対応をそのまま書く

最終的なテストは、プロパティと期待値を1行ずつ対応させた。

```csharp
[Fact]
public void ModelValidationReason_DocumentedReasons_ExposeExpectedValues()
{
    Assert.Equal("CONTROL_CHARACTER", ModelValidationReason.ControlCharacter.Value);
    Assert.Equal("INVALID_HEX_COLOR", ModelValidationReason.InvalidHexColor.Value);
    Assert.Equal("LENGTH_OUT_OF_RANGE", ModelValidationReason.LengthOutOfRange.Value);
    Assert.Equal("REQUIRED", ModelValidationReason.Required.Value);
    Assert.Equal("UNSUPPORTED_IMAGE_TYPE", ModelValidationReason.UnsupportedImageType.Value);
    Assert.Equal("VISIBLE_CHARACTER_REQUIRED", ModelValidationReason.VisibleCharacterRequired.Value);
}
```

この形には重複があるが、テストの入力、対象、期待値を追跡する処理がない。失敗時には、どの公開値が不一致なのかを該当行から確認できる。

## なぜ「すべて」を自動列挙しなかったのか

reflectionで`.GetProperties()`を呼び、LINQで値を並べ替えて期待配列と比較すれば、現在の公開プロパティ数を機械的に検査できる。ただし、テスト側に次の知識が増える。

- 対象プロパティを選別する規則
- staticプロパティから値を取得する方法
- 列挙順を揃える並べ替え規則
- 実装値と期待値を別々に保守する対応関係

今回の値は6件であり、直接assertionの方が仕様表との対応を短く読めた。この判断は「テストではLINQを使わない」という一般則ではない。大量の同型データ、Theory、組み合わせ検証など、データ変換自体が単純で再利用価値を持つ場合にはLINQやデータ駆動テストが適する。

## テスト名と仕様IDを実際の契約へ合わせる

直接assertionのテストには、新しいIDとして`REASON-01`を付けた。元の`ERROR-02`は、未定義値の生成拒否という別の契約を表すためである。

```csharp
// ID: REASON-01
// Source: docs/v1/api/models.md §12 Domain Error Examples.
// Given: each documented ModelValidationReason property
// When: its machine-readable value is read
// Then: it matches the corresponding documented error code
```

テスト本文だけでなく、ID、参照先、Given／When／Thenも実際のassertionへ合わせることで、仕様からテストへの追跡可能性を保った。

## 検証結果

2026年8月11日に`ModelValidationReasonTests`だけを実行し、成功1件、Skipped 1件、失敗0件を確認した。全体のテストでは成功8件、Skipped 33件、失敗0件だった。

```powershell
dotnet test backend\Mojica.Api.Tests\Mojica.Api.Tests.csproj `
  --no-restore `
  --filter "FullyQualifiedName~ModelValidationReasonTests" `
  --verbosity minimal
```

Skippedの1件は`ERROR-02`である。これは失敗ではなく、適切なテスト手段が未確定であることを明示した状態である。

## 適用するときの判断基準

公開カタログのテストでは、次の順で考えると整理しやすい。

1. まず、テストが証明したい契約を1文にする。
2. 存在する値の対応と、存在しない生成経路の保証を分ける。
3. 少数の固定値なら、プロパティと期待値を直接assertionする。
4. reflectionを使う場合は、検査規則そのものが安定した公開契約かを確認する。
5. 適切に証明できない契約は、別IDのSkipped計画として可視化する。

## まとめ

テストコードも保守対象であり、処理を高度にするほど良いわけではない。Mojicaでは、reflectionによる生成経路検査と、仕様にある公開値の確認を別の問いへ分けた。現在確認できる6件は直接assertionで読みやすく固定し、未定義値を生成できないという契約は未実装のまま正直に残した。

重複を減らすことより、テストを読んだ人が「何が期待値で、何を検査しているか」をすぐ理解できることを優先した例である。

## 根拠

- `docs/v1/api/models.md` §11-12
- `backend/Mojica.Api.Tests/Models/ModelValidationReasonTests.cs`
- `9fe7fef test: guard validation reason factories`
- `b3eedac test: inspect all reason creation paths`
- `9016e14 refactor: simplify validation reason test`
- `ce58d37 refactor: read validation reasons directly`
- `ef841b0 test: align validation reason contract comment`
- `c0d8b63 test: separate validation reason contracts`

