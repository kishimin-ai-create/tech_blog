# C#で閉じたImageTypeをTryCreateとして実装する

APIが受け取る画像種別を生の文字列のまま運ぶと、未検証値がDomain層より先へ流れやすい。Mojicaでは、仕様で許可された3種類だけを表せる`ImageType`を作り、生成時に未定義値をエラーへ変換した。

## 任意の文字列とDomain valueを分ける

Mojicaが受け付ける画像種別は`standard`、`x-background`、`x-icon`の3つである。HTTP DTOでは文字列を受け取るが、Domain層より内側では検証済みの値として扱いたい。そこで、コンストラクターをprivateにした`sealed record`へ定義済み値を集約した。

```csharp
public sealed record ImageType
{
    public static ImageType Standard { get; } = new("standard");
    public static ImageType XBackground { get; } = new("x-background");
    public static ImageType XIcon { get; } = new("x-icon");

    private ImageType(string value)
    {
        Value = value;
    }

    public string Value { get; }
}
```

型の外から`new ImageType("other")`を呼べないため、生成済みの値は3種類のいずれかになる。

## TryCreateで成功値とDomain errorを返す

入力文字列からの変換は`TryCreate`へ置いた。

```csharp
imageType = value switch
{
    "standard" => Standard,
    "x-background" => XBackground,
    "x-icon" => XIcon,
    _ => null,
};

error = imageType is null
    ? new ModelValidationError(
        "type",
        ModelValidationReason.UnsupportedImageType)
    : null;

return imageType is not null;
```

例外ではなくboolとout引数を使うのは、未定義の画像種別が予期しない実行時障害ではなく、分類可能な入力エラーだからである。失敗時には`target`を`type`、理由を`UNSUPPORTED_IMAGE_TYPE`として上位層へ渡せる。

## 成功入力はTheoryで確認する

3種類には同じ生成契約を適用するため、xUnitの`Theory`と`InlineData`を使った。テストでは成功フラグ、生成値、保持した文字列、エラーがないことを直接確認する。失敗ケースではboolだけでなく、生成値がnullであることと、エラーコード、対象、理由も検証した。

## 適用範囲

この設計が保証するのは、`TryCreate`から得た`ImageType`が定義済み値であることだ。HTTPリクエストで`type`が欠落した場合の`REQUIRED`判定や、画像種別から外部APIのendpointを選ぶ処理は別の境界に残している。

## 検証結果

2026年8月11日に`ImageTypeTests`を実行し、成功4件、失敗0件、Skipped 0件を確認した。全体テストは成功12件、失敗0件、Skipped 30件だった。

## まとめ

外部入力の文字列をそのままDomain値として扱わず、定義済み値と変換処理を1つの型へ集約すると、後続処理は検証済みの画像種別を受け取れる。閉じた生成経路は型構造で表し、Unit testでは公開されている`TryCreate`の成功と失敗を確認する。

## 根拠

- `docs/v1/api/models.md` §4、§12、§13
- `backend/Mojica.Api/Models/ImageType.cs`
- `backend/Mojica.Api.Tests/Models/ImageTypeTests.cs`
- `0becf3c test: define supported image types`
- `00e2079 feat: add supported image types`
- `a8497ee test: reject unsupported image types`
- `016120d feat: reject unsupported image types`

