# C#で検証理由を閉じた値集合として表現する

APIの検証エラーでは、表示文言とは別に、プログラムが安定して判定できる理由が必要になる。本記事では、Mojicaで文字列コードを`ModelValidationReason`という閉じた値集合へまとめ、Domain ModelからHTTPやローカライズの関心を分離した変更を説明する。

## 問題：生の文字列は未定義値を表現できる

Mojicaの設計書は、検証エラーの`reason`を機械判定可能な値とし、表示メッセージをDomain Modelへ含めないと定めている。仕様上の理由は次の6種類である。

| プロパティ | 機械可読値 |
| --- | --- |
| `Required` | `REQUIRED` |
| `LengthOutOfRange` | `LENGTH_OUT_OF_RANGE` |
| `ControlCharacter` | `CONTROL_CHARACTER` |
| `InvalidHexColor` | `INVALID_HEX_COLOR` |
| `UnsupportedImageType` | `UNSUPPORTED_IMAGE_TYPE` |
| `VisibleCharacterRequired` | `VISIBLE_CHARACTER_REQUIRED` |

呼び出し側が任意の文字列を渡せる設計では、タイプミスや仕様外の値もコンパイルを通る。そこで、生成可能な値を型の公開プロパティへ集約した。

## sealed recordとprivateコンストラクターを使う

実装は、公開された静的プロパティとprivateコンストラクターから成る。

```csharp
public sealed record ModelValidationReason
{
    public static ModelValidationReason ControlCharacter { get; }
        = new("CONTROL_CHARACTER");

    public static ModelValidationReason Required { get; }
        = new("REQUIRED");

    private ModelValidationReason(string value)
    {
        Value = value;
    }

    public string Value { get; }
}
```

`sealed`により派生型から値を増やせず、privateコンストラクターにより型の外から任意値を生成できない。`record`なので、同じ値を表すインスタンスには値指向の等価性が与えられる。

この形式は言語組み込みの`enum`とは異なり、外部へ公開する文字列値をそのまま保持できる。HTTPレスポンスへの変換方法や表示文言は、この型の責務に含めていない。

## 仕様にある値を一度に追加する

変更前は`Required`だけが実装されていた。変更後は`docs/v1/api/models.md` §12にある残り5種類を追加し、仕様表とプロダクションコードの対応を揃えた。

ここで確認できた事実は「現在の6プロパティが仕様書と一致する」ことである。将来も値が増えないことや、外部のバイナリ互換性まで保証したわけではない。

## 公開値を直接テストする

各プロパティと文字列値の対応は、xUnitで直接比較した。

```csharp
Assert.Equal("CONTROL_CHARACTER", ModelValidationReason.ControlCharacter.Value);
Assert.Equal("INVALID_HEX_COLOR", ModelValidationReason.InvalidHexColor.Value);
Assert.Equal("LENGTH_OUT_OF_RANGE", ModelValidationReason.LengthOutOfRange.Value);
Assert.Equal("REQUIRED", ModelValidationReason.Required.Value);
Assert.Equal("UNSUPPORTED_IMAGE_TYPE", ModelValidationReason.UnsupportedImageType.Value);
Assert.Equal("VISIBLE_CHARACTER_REQUIRED", ModelValidationReason.VisibleCharacterRequired.Value);
```

期待値を文字列リテラルで置くため、プロダクションコードが誤った値へ変わればテストは失敗する。実装から期待値を導出して自己比較するテストにはしていない。

## 検証結果

2026年8月11日に、対象テスト、Releaseビルド、全テストとcoverageを確認した。

```powershell
dotnet test backend\Mojica.Api.Tests\Mojica.Api.Tests.csproj `
  --no-restore `
  --filter "FullyQualifiedName~ModelValidationReasonTests" `
  --verbosity minimal

dotnet build backend\Mojica.Backend.sln `
  --configuration Release `
  --no-restore
```

対象テストは成功1件、Skipped 1件、失敗0件だった。Releaseビルドは警告0件、エラー0件で成功した。全体では成功8件、Skipped 33件、失敗0件で、line coverageは96.72%、branch coverageは100%だった。

## 制約と次の課題

`ERROR-02`として計画された「未定義の理由を生成できないこと」のテストはSkippedのままである。通常のユニットテストで存在しない公開APIを証明しようとすると、reflectionで型構造を詳細に検査するテストになりやすい。必要性が高まった場合は、コンパイラーまたはAPI互換性検査の境界で保証する余地がある。

また、この変更は検証理由の語彙を用意しただけであり、`ImageType`や文字列Value Objectの具体的な検証処理は実装していない。

## まとめ

外部契約で使う検証理由は、生の文字列として各所へ散らすのではなく、生成経路を限定した型へ集約できる。Mojicaでは、`sealed record`、privateコンストラクター、静的プロパティを組み合わせ、仕様書にある6種類の機械可読値を表現した。

重要なのは、閉じた型を作ること自体ではなく、Domain Modelには機械判定可能な理由だけを置き、表示文言やHTTP変換を別の境界へ残すことである。

## 根拠

- `docs/v1/api/models.md` §11-12
- `backend/Mojica.Api/Models/ModelValidationReason.cs`
- `backend/Mojica.Api.Tests/Models/ModelValidationReasonTests.cs`
- `62496e4 test: define closed validation reasons`
- `808ea38 feat: add closed validation reasons`

