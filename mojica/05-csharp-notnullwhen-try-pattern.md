# NotNullWhenでTryパターンのout引数を安全に使う

C#でboolとout引数を返すTryパターンを自作すると、実行時には値が正しくても、コンパイラーが成功後のout値をnullableと判断することがある。Mojicaの`ImageType.TryCreate`では、`NotNullWhen`で戻り値とnull状態の関係を公開契約へ追加した。

## 実装値だけではnullable解析へ伝わらない

最初のシグネチャは次の形だった。

```csharp
public static bool TryCreate(
    string value,
    out ImageType? imageType,
    out ModelValidationError? error)
```

実装は成功時に`imageType`、失敗時に`error`を必ず代入する。しかし、呼び出し側のコンパイラーはメソッド本体を追跡してboolとの対応を推測しない。

```csharp
if (ImageType.TryCreate(value, out var imageType, out var error))
{
    Console.WriteLine(imageType.Value);
}
```

属性がない状態で警告をエラーとしてビルドすると、`imageType.Value`に`CS8602`が発生した。失敗分岐で`error.Code`を読む場合も同じだった。

## NotNullWhenで条件付き契約を表す

`System.Diagnostics.CodeAnalysis.NotNullWhenAttribute`は、bool戻り値が指定値のとき、その引数が非nullであるとnullableフロー解析へ伝える。

```csharp
public static bool TryCreate(
    string value,
    [NotNullWhen(true)] out ImageType? imageType,
    [NotNullWhen(false)] out ModelValidationError? error)
```

戻り値が`true`なら`imageType`、`false`なら`error`を非nullとして扱える。これは実行時チェックを追加する属性ではないため、実装が宣言どおりの値を返す責任は残る。成功時の`imageType`と失敗時の`error`は通常のUnit testでも確認する。

## nullable契約とUnit testの責務を分ける

一時的に、テスト本文をbool分岐へ変更して属性の有無を警告エラービルドで検出した。しかし、通常の振る舞いテストとしては、成功フラグとout値を直接assertionする方が読みやすい。

最終的には属性をプロダクションコードへ残し、テストは実行時の観測可能な結果を表す形へ戻した。属性そのものの回帰を継続的に強制する必要がある場合は、警告をエラーとして扱うCIビルドやAPI解析を別途品質ゲートにする方法がある。

## 検証結果

修正前の再現では、成功側と失敗側の2か所で`CS8602`が発生した。属性追加後は次のコマンドが警告0件、エラー0件で成功した。

```powershell
dotnet build backend/Mojica.Api.Tests/Mojica.Api.Tests.csproj `
  --no-restore `
  -warnaserror
```

## まとめ

Tryパターンの実装では、値を正しく代入するだけでなく、戻り値とout引数のnull状態をシグネチャで表す必要がある。`NotNullWhen`を使えば、呼び出し側は`!`による抑制なしに安全な分岐を書ける。一方、Unit testは実行時の振る舞いを読みやすく表し、コンパイル時契約の検査とは責務を分ける。

## 根拠

- `backend/Mojica.Api/Models/ImageType.cs`
- `backend/Mojica.Api.Tests/Models/ImageTypeTests.cs`
- `c4cc501 test: require image type null-state contracts`
- `13f472c fix: describe image type null-state outputs`
- `e4cdb7e refactor: simplify image type assertions`
