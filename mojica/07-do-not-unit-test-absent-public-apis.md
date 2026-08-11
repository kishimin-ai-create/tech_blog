# 存在しない公開APIをUnit testで証明しない

「任意値を生成する公開経路がないこと」をUnit testで保証しようとすると、reflectionで型の全メンバーを調べる実装になりやすい。Mojicaでは、この検査を通常のランタイムテストの責務から外し、存在しないAPIを表すSkipped計画も削除した。

## 公開コンストラクターだけでは不十分だった

閉じた`ModelValidationReason`に任意値の生成経路がないことを確認するため、最初はpublic constructorの有無を調べる案があった。しかし、生成経路はコンストラクターだけではない。

- static factory method
- 対象型を返すinstance method
- implicit／explicit conversion operator
- recordが生成する特殊メンバー

検査対象を増やすほど、テスト側でC#とreflection APIのメンバー分類を再実装することになる。それでも、将来追加されるすべての表現を漏れなく検出できる保証にはならない。

## テスト名の保証と実際の検査範囲がずれる

例えば`CannotRepresentUndefinedReason`という名前は、任意の生成経路がないという強い保証に見える。ところが、テストがstaticな通常メソッドだけを列挙していれば、instance methodや変換演算子を見落とす。

この状態ではテストがGreenでも、主張した契約を証明できていない。検査条件を複雑にして偽の安心を増やすより、Unit testが確認できる範囲を公開された操作と観測可能な結果へ限定する方が正確である。

## 型構造とランタイム契約の責務を分ける

Mojicaでは、閉じた値集合をprivateコンストラクター、`sealed`型、定義済みの静的プロパティ、コンパイラー、コードレビューで維持する。

Unit testでは、実際に呼び出せる`ImageType.TryCreate`について、定義済み値が成功し、未定義値が失敗することを検証する。存在しないconstructorやfactoryをreflectionで探すテストは置かない。

公開API互換性を自動検査する必要が将来生じた場合は、専用のAPI analyzerやコンパイル時検査を別の意思決定として導入できる。

## Skipped計画も残さない

`ERROR-02`は、未定義の`ModelValidationReason`を作れないことを表すSkippedテストだった。しかし、通常のUnit testへ変換しないと決めた後も残すと、永続的な未完了テストに見える。

そこで、ADR 0019の決定に従い、属性、メソッド、計画コメントを含めて削除した。これは契約を捨てたのではなく、保証する責務を型システムとレビューへ移した結果である。

## 直接確認できる公開値はテストする

一方、現在公開されている6つの`ModelValidationReason`と文字列値の対応は、利用者から観測できる。こちらはreflectionを使わず、直接assertionで検証している。

```csharp
Assert.Equal(
    "UNSUPPORTED_IMAGE_TYPE",
    ModelValidationReason.UnsupportedImageType.Value);
```

「存在する値が正しいか」と「存在しないAPIがないか」を別の問いとして扱うことが重要である。

## まとめ

Unit testは、呼び出せる操作と観測できる結果を得意とする。存在しない公開生成経路をランタイムで証明しようとすると、言語仕様の不完全な再実装になりやすい。閉じた型の構造制約は型システムとレビューへ任せ、公開された生成操作の成功・失敗だけをUnit testで確認する。

## 根拠

- ADR 0019「存在しない公開生成経路をランタイムテストで証明しない」
- `backend/Mojica.Api/Models/ModelValidationReason.cs`
- `backend/Mojica.Api.Tests/Models/ModelValidationReasonTests.cs`
- `73d3772 test: remove absent API test plan`
