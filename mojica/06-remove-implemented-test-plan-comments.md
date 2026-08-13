# 実装済みxUnitテストから計画コメントを外す

## はじめに

テスト実装前の`ID`、`Given`、`When`、`Then`は、作業対象を明確にする。しかし、実装後も同じ説明を残すと、テスト名、入力、操作、assertionとコメントの二重管理になる。Mojicaでは、実装済みテストだけから計画コメントを削除した。

## 計画コメントには有効期間がある

実装前の足場は、次の情報をコメントで保持していた。

```csharp
// ID: IMGTYPE-01
// Source: docs/v1/api/models.md §4 ImageType.
// Given: each supported value
// When: ImageType creation is requested
// Then: creation succeeds
// Priority: High
```

この段階ではassertionがないため、コメントが将来実装する契約を表す。一方、実装後のテストには、テスト名、`InlineData`、対象メソッドの呼び出し、assertionとして同じ情報が存在する。両方を残すと、仕様変更時にコードだけ、またはコメントだけが更新される可能性がある。

## 削除対象をラベルで限定する

コメント全般を削除したわけではない。対象は、テストメソッド内で連続する`ID`、`Source`、`Given`、`When`、`Then`、`Error`、`Blocked by`、`Priority`の計画用ラベルだけに限定した。

実装理由、採用しなかった方法、境界条件、analyzer抑制などの「コードから読み取れない理由コメント」は保護する。また、本文が計画コメントだけのSkippedテストは、コメントだけを消すと空になるため、コメント整理とは別の判断として扱う。

## 差分では実行コードが不変であることを確認する

Mojicaでは、`ImageTypeTests`と`GeneratedImageTests`から計画ブロックを削除した。削除後に差分を確認し、属性、テストデータ、メソッド名、入力値、対象メソッドの呼び出し、assertionが変わっていないことを確かめた。

つまり、変更はテストの振る舞いではなく、実装済み契約の重複表現だけである。

## Skipped計画は別の扱いにする

未実装のSkippedテストでは、計画コメントが唯一の本文である。これを機械的に削除してはいけない。まず、そのテストを今後実装するのか、自動テストの責務外としてメソッドごと削除するのかを決める必要がある。

Mojicaの`ERROR-02`は後者だった。存在しない公開生成経路をUnit testで証明しないというADRに基づき、コメント削除とは別の変更としてSkippedメソッド全体を削除した。

## 動作確認

コメント整理後の全体テストは成功12件、失敗0件、Skipped 30件だった。実装済みテストの属性、データ、assertionは変更していない。

## まとめ

計画コメントは実装前の追跡に役立つが、実装後はテストコード自体を契約の一次表現にできる。安全に整理するには、計画ラベルだけを対象にし、理由コメントと未実装のSkipped計画を区別する。コメントを減らすことではなく、同じ契約を二重管理しないことが目的である。

## 根拠

- `backend/Mojica.Api.Tests/Models/ImageTypeTests.cs`
- `backend/Mojica.Api.Tests/Models/GeneratedImageTests.cs`
- `backend/Mojica.Api.Tests/Models/ModelValidationReasonTests.cs`
- `ba1cf68 test: remove redundant test plan comments`
- `2204111 test: remove generated image plan comments`
- `$HOME/.codex/skills/remove-xunit-test-plan-comments/SKILL.md`
