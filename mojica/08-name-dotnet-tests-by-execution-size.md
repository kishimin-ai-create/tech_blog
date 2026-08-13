# .NETのテストファイル名でSmall・Mediumを識別する

## はじめに

テストがUnitかIntegrationかという目的だけでは、実行コストや依存範囲をファイル一覧から判断しにくい。Mojicaでは、テストファイル名とクラス名に`Small`または`Medium`を含め、実行サイズを明示した。

この記事では、分類基準と、既存テストの振る舞いを変えずに命名を移行した結果を記録する。

## 前提・環境

- Runtime: .NET 8
- Test Framework: xUnit 2.5.3
- 対象: ASP.NET Core APIのModelテストとHTTPエンドポイントテスト

## やってみた結果

外部プロセスを使わずModel内で完結するテストを`*SmallTests.cs`、`WebApplicationFactory<Program>`でアプリを起動してHTTP境界を通るテストを`*MediumTests.cs`へ変更した。

たとえば、`RgbColorSmallTests`は整数入力と戻り値だけを扱う。一方、`HealthEndpointMediumTests`はASP.NET Coreのテストサーバーと`HttpClient`を使用する。この依存範囲の差がファイル名から分かるようになった。

## 実装・検証

### Step 1: 依存範囲で分類する

Smallは単一プロセスかつ外部I/Oなし、Mediumは複数コンポーネントを接続するが管理されたローカル環境で完結する分類とした。

```text
GeneratedImageTests.cs → GeneratedImageSmallTests.cs
RenderTextTests.cs     → RenderTextSmallTests.cs
HealthEndpointTests.cs → HealthEndpointMediumTests.cs
```

ファイル名だけでなく、C#のクラス名も同じサイズ名へ揃えた。

### Step 2: 振る舞いを変えずに名前だけを変更する

サイズ移行ではテスト内容を変更しなかった。Gitは9ファイルをrenameとして認識し、後続のRGB実装は`RgbColorSmallTests.cs`上で進めた。

最終確認では次のコマンドを実行した。

```powershell
dotnet test backend/Mojica.Backend.sln
```

結果は41件成功、11件Skip、失敗0件だった。Skipは`HexColor`と`ImageGenerationRequest`の既存実装計画である。

## つまずいたところ

この命名変更そのものでは、実行を妨げる問題は発生しなかった。

## 学んだこと

Unit／Integrationはテストの目的を説明し、Small／Medium／Largeは依存範囲と実行コストを説明する。両者を同義語として扱わず、リポジトリの実行規約をファイル名へ反映すると、テストを読む前にフィードバック速度を判断しやすくなる。

## まとめ

Mojicaでは、外部I/OのないModelテストをSmall、ASP.NET CoreのHTTP境界を通るテストをMediumとして命名した。移行は名前に限定し、全体テストで既存の振る舞いが維持されることを確認した。

## 参考資料

- Mojica ADR 0004「Repository-defined test size naming」
- Mojica commit `8e69f36 test: identify test files by size`

