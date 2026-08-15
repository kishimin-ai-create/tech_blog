# C#のTryCreateで範囲外のRGB値を生成させない

## はじめに

RGB色の各成分には0から255までという境界がある。任意の`int`をそのままModelへ保存すると、不正な値が後続の画像生成処理まで流れる可能性がある。

Mojicaでは、privateコンストラクターと`TryCreate`を組み合わせ、検証済みの色だけを`RgbColor`として表現した。この記事では成功値と検証エラーを同時に扱う契約を説明する。

## 前提・環境

- Runtime: .NET 8
- Language: C#（Nullable有効）
- Test Framework: xUnit 2.5.3

## やってみた結果

`RgbColor.TryCreate`は、全成分が0から255なら`true`と生成済みオブジェクトを返す。いずれかが範囲外なら`false`を返し、色は生成せず、どの成分がどの境界を外れたかを`ModelValidationError`で通知する。

```csharp
var succeeded = RgbColor.TryCreate(
    red,
    green,
    blue,
    out var color,
    out var error);
```

## 実装・検証

### Step 1: 有効値だけを生成する

コンストラクターをprivateにして、公開生成経路を`TryCreate`へ限定した。

```csharp
if (TryGetRangeError("red", red, out error)
    || TryGetRangeError("green", green, out error)
    || TryGetRangeError("blue", blue, out error))
{
    color = null;
    return false;
}

color = new RgbColor(red, green, blue);
error = null;
return true;
```

検証は`red`、`green`、`blue`の順に行うため、複数成分が不正なら最初の成分が返る。

### Step 2: 機械判定できる範囲エラーを返す

範囲外の値には`VALUE_OUT_OF_RANGE`を使用し、属性名と範囲を構造化した。

```csharp
new ModelValidationError(
    target,
    ModelValidationReason.ValueOutOfRange,
    new Dictionary<string, string>
    {
        ["minimum"] = "0",
        ["maximum"] = "255",
        ["actual"] = value.ToString(CultureInfo.InvariantCulture),
    });
```

表示文ではなくコードと詳細を返すため、呼び出し側は言語別メッセージへ変換できる。

### Step 3: 境界と各成分をTheoryで検証する

有効値は下限、上限、中間値を含む3件で確認した。無効値は`-1`と`256`について、赤・緑・青を個別に検証した。合計9件の`RgbColorSmallTests`が、成功結果、値の保持、生成失敗、対象成分、エラーコード、境界詳細を確認する。

```powershell
dotnet test backend/Mojica.Api.Tests/Mojica.Api.Tests.csproj `
  --filter "FullyQualifiedName~RgbColorSmallTests"
```

結果は9件成功、失敗0件、Skip 0件だった。Release構成で収集した全体カバレッジはLine 96.47%、Branch 100%だった。

## つまずいたところ

### 問題

対象テストと全体テストを並列実行したところ、両方が同じ`obj`配下のStatic Web Assetsキャッシュへ書き込み、`IOException`になった。

### 原因

同一プロジェクトに対する2つの`dotnet test`が、同じ中間生成ファイルを同時に使用したためである。RGB実装やテストの失敗ではなかった。

### 解決

対象テストと全体テストを直列に実行した。どちらも成功し、以後このリポジトリでは同一プロジェクトへの`dotnet`コマンドを並列実行しない運用に切り替えた。

## 学んだこと

値オブジェクトは値を保持するだけでなく、「不正な状態を生成できない」という境界を担当できる。Tryパターンでは、成功時の値と失敗時の理由を排他的に返し、`NotNullWhen`でその関係をNullable解析へ伝えられる。

また、境界値だけでなく各成分をTheoryへ並べることで、検証対象名の取り違えも検出できる。

## まとめ

privateコンストラクターと`TryCreate`により、0から255までのRGBだけをModelとして生成できるようにした。範囲外では`VALUE_OUT_OF_RANGE`と構造化された詳細を返し、xUnitの9ケースで公開契約を固定した。

## 参考資料

- Mojica `docs/v1/api/models.md` §8 RgbColor
- Mojica commits `a8f605c`〜`7e0d087`
