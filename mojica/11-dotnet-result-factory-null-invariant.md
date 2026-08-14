# Result factoryがnullから不正状態を作る問題の原因と解決方法

## 結論

nullable reference typeの注釈だけでは、実行時に`Success(null!)`や`Failure(null!)`を拒否できない。成功・失敗Resultの必須値はpublic factoryで検証し、不正なvariantを生成境界で拒否する。

## 発生した問題

### 症状

`ImageGenerationPortResult`は`Error is null`から成功を判定していたが、factoryの引数を実行時に検証していなかった。

```csharp
ImageGenerationPortResult.Success(null!);
ImageGenerationPortResult.Failure(null!);
```

修正前はどちらも例外を出さなかった。前者はDataを持たない成功になり、後者は失敗factoryから`IsSuccess == true`の空Resultになった。

### 発生条件

- Environment: Windows、PowerShell
- Runtime: .NET 8
- Configuration: nullable reference type有効

null forgiving operatorを使用する場合や、nullable警告を契約保証として利用できない呼び出し境界で発生する。

### 影響

Resultを受け取るServiceが「成功ならDataがある」「失敗ならErrorがある」と仮定できず、後段でnullを処理する必要が生じる。特に`Failure(null!)`は失敗を成功として分類するため、variant判定そのものを壊す。

## 調査

### 仮説1

非nullable引数なら実行時にもnullが拒否されると考えられるが、C#のnullable reference typeは主に静的解析である。公開factoryへnullを渡す回帰テストを追加した。

```csharp
Assert.Throws<ArgumentNullException>(() =>
    ImageGenerationPortResult.Success(null!));
```

実測結果は`No exception was thrown`であり、修正前のfactoryが実行時契約を持たないことを確認した。

### 仮説2

失敗factoryも同じ問題を持つか、別のテストで確認した。

```csharp
Assert.Throws<ArgumentNullException>(() =>
    ImageGenerationPortResult.Failure(null!));
```

こちらも修正前は例外を出さなかった。`IsSuccess`が`Error is null`で決まるため、派生症状として空Resultが成功扱いになることもコードから確認した。

## 原因

public factoryの引数へ非nullable注釈を付けただけで、Resultの不変条件を実行時に検証していなかったことが原因である。注釈は通常のC#呼び出しで警告を提供するが、`null!`や警告設定の異なる呼び出し元からnullが渡されることを防がない。

## 解決方法

各factoryで、そのvariantに必須の値を検証した。

```csharp
public static ImageGenerationPortResult Success(GeneratedImageData data)
{
    ArgumentNullException.ThrowIfNull(data);
    return new ImageGenerationPortResult(data, null);
}

public static ImageGenerationPortResult Failure(ImageGenerationPortError error)
{
    ArgumentNullException.ThrowIfNull(error);
    return new ImageGenerationPortResult(null, error);
}
```

不正状態がResult内部へ入る前に拒否されるため、利用側はvariant判定後の値の存在を前提にできる。

## 動作確認

```text
dotnet test backend/Mojica.Api.Tests/Mojica.Api.Tests.csproj --no-restore --filter "FullyQualifiedName~ImageGenerationPortResultSmallTests"

Passed: 2, Failed: 0
```

全体ではRelease buildが警告0件・エラー0件、テストは81件成功、2件Skip、失敗0件だった。Coverletの全体結果はLine 95.95%、Branch 94.11%だった。

## 再発防止

- 成功・失敗factoryごとに、必須値がnullの場合の公開挙動をテストする。
- Resultのtruth tableを作り、成功値と失敗値の許される組み合わせを確認する。
- nullable注釈と実行時の不変条件を別の保証として扱う。
- 判断はADR-0026「Result variantの不変条件を公開factoryで保証する」に記録した。

## まとめ

- 症状: Dataのない成功と、成功扱いされる空の失敗Resultを作成できた。
- 原因: nullable注釈だけに依存し、public factoryで必須値を検証していなかった。
- 解決方法: `ArgumentNullException.ThrowIfNull`で各variantの必須値を生成境界で拒否した。
- 得られた教訓: Resultの不変条件は、利用側ではなく公開生成経路で保証する。
