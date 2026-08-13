# C# recordの辞書プロパティを内容で値比較する

## 結論

C#の`record`は値指向の型を簡潔に定義できる。しかし、プロパティに辞書を持たせただけでは、辞書の要素まで自動的に値比較されるわけではない。本記事では、Mojicaの`ModelValidationError`で、辞書の内容と挿入順を考慮した等価性を実装した方法を説明する。

## 問題：recordでも辞書は内容比較にならない

Mojicaの検証エラーは、対象、理由、補足情報をDomain valueとして保持する。

```csharp
public sealed record ModelValidationError
{
    public string Target { get; }

    public ModelValidationReason Reason { get; }

    public IReadOnlyDictionary<string, string> Details { get; }
}
```

recordが生成する等価比較は、各メンバーの等価性を利用する。`ReadOnlyDictionary<TKey, TValue>`は辞書要素の構造的な値等価性を提供しないため、内容が同じでも別々に生成した辞書を持つrecord同士は、そのままでは期待するDomain valueの等価性にならない。

必要な契約は次のとおりだった。

- `Target`と`Reason`が等しい
- `Details`のキーと値がすべて等しい
- `Details`の挿入順は等価性へ影響しない
- 等しいオブジェクトは同じハッシュ値を返す

## 先に期待する振る舞いをテストする

実装前に、別々の辞書を異なる順番で作ってもエラーが等しいことをxUnitで固定した。

```csharp
var first = new ModelValidationError(
    "text",
    ModelValidationReason.Required,
    new Dictionary<string, string>
    {
        ["minimumLength"] = "1",
        ["actualLength"] = "0",
    });

var second = new ModelValidationError(
    "text",
    ModelValidationReason.Required,
    new Dictionary<string, string>
    {
        ["actualLength"] = "0",
        ["minimumLength"] = "1",
    });

Assert.Equal(first, second);
Assert.Equal(first.GetHashCode(), second.GetHashCode());
```

このテストでは「同じ辞書インスタンス」を再利用していない。参照の一致ではなく、内容の一致を検証するためである。

## Equalsでキーと値を比較する

`ModelValidationError`では、型固有の`Equals`を実装した。

```csharp
public bool Equals(ModelValidationError? other)
{
    return ReferenceEquals(this, other)
        || other is not null
        && string.Equals(Target, other.Target, StringComparison.Ordinal)
        && Equals(Reason, other.Reason)
        && Details.Count == other.Details.Count
        && Details.All(detail =>
            other.Details.TryGetValue(detail.Key, out var value)
            && string.Equals(detail.Value, value, StringComparison.Ordinal));
}
```

比較は次の順序で行う。

1. 同じインスタンスなら直ちに`true`を返す。
2. 比較対象が`null`なら`false`を返す。
3. `Target`と`Reason`を比較する。
4. 辞書の要素数を比較する。
5. すべてのキーが比較対象にも存在し、値が一致することを確認する。

要素数を先に確認するため、一方が余分なキーを持つ場合も等しいとは判定されない。キー検索を使うので、挿入順は比較結果へ影響しない。

## GetHashCodeでは順序を正規化する

等価なオブジェクトは同じハッシュ値を返す必要がある。辞書を列挙した順番のままハッシュへ追加すると、挿入順が違うだけで結果が変わり得る。そのため、キーをordinal順に並べてからハッシュを生成する。

```csharp
public override int GetHashCode()
{
    var hash = new HashCode();
    hash.Add(Target, StringComparer.Ordinal);
    hash.Add(Reason);

    foreach (var detail in Details.OrderBy(
                 detail => detail.Key,
                 StringComparer.Ordinal))
    {
        hash.Add(detail.Key, StringComparer.Ordinal);
        hash.Add(detail.Value, StringComparer.Ordinal);
    }

    return hash.ToHashCode();
}
```

ここでのソートは、辞書の内容を変更するためではなく、同じ内容を常に同じ順番でハッシュへ入力するための正規化である。

## 可変な入力辞書をそのまま保持しない

等価性を安定させるには、生成後に`Details`が外部から変化しないことも重要になる。コンストラクターでは入力辞書をコピーしてから読み取り専用にした。

```csharp
Details = details is null
    ? EmptyDetails
    : new ReadOnlyDictionary<string, string>(
        new Dictionary<string, string>(details));
```

`ReadOnlyDictionary`で元の辞書を包むだけでは、元の辞書を持つ呼び出し元が内容を変更できる。先に`new Dictionary`でコピーすることで、生成時点のスナップショットを保持できる。

## 非等価ケースも確認する

等価ケースだけでは、比較条件の不足を検出しにくい。Mojicaでは次の差分もテストした。

- 比較対象が`null`
- `Target`が異なる
- `Details`の有無が異なる
- キーが異なる
- 同じキーの値が異なる

これにより、常に`true`を返す実装や、一部のプロパティを比較し忘れた実装を防げる。

## 比較結果

2026年8月11日に次のコマンドを実行した。

```powershell
dotnet test backend\Mojica.Backend.sln --no-restore --verbosity minimal
```

結果は失敗0件、成功7件、Skipped 33件、合計40件だった。Skippedは、今回の対象外である他のDomain Modelのテスト計画である。

## まとめ

recordの値等価性は、メンバー自身の等価性に依存する。辞書を含むDomain valueでは、次の3点を明示的に設計する必要がある。

1. `Equals`で辞書のキーと値を比較する。
2. `GetHashCode`では列挙順を正規化する。
3. 入力辞書をコピーし、生成後の値を安定させる。

コレクションを持つrecordを「値」として扱うときは、型をrecordにしただけで完了とせず、コレクションの等価性と可変性までテストで確認することが重要である。

## 根拠

- `backend/Mojica.Api/Models/ModelValidationError.cs`
- `backend/Mojica.Api.Tests/Models/ModelValidationErrorTests.cs`
- `f661249 test: define validation error equality`
- `ecb3ea9 fix: compare validation detail values`
- `4be284a test: cover validation error inequality`
