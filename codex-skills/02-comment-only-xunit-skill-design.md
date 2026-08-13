# xUnitのテストケースをコードではなくコメントだけで設計するSkill

## 結論

テストケースの洗い出しだけが必要な場面で、AIが`[Fact]`や空のテストメソッドまで生成すると、レビュー対象と実装済み範囲が曖昧になる。本記事では、ASP.NET Core向けのxUnitテスト設計Skillを「コメントだけを書き、テストコードは書かない」契約へ変更した方法を説明する。

対象読者は、CodexのユーザーSkillを管理し、テスト実装前の計画を既存の`.cs`ファイルへ残したい開発者である。

## 受け入れ条件を先に固定する

今回の要求は「テストコードは絶対に書かず、コメントだけを書く」ことだった。この要求を曖昧な推奨ではなく、Skillの制約として定義した。

許可する書き込みは、`//`で始まるコメントだけである。次の要素は生成・変更しない。

- `[Fact]`と`[Theory]`
- テストメソッドとテストクラス
- fixture、fake、stub
- assertionとデータプロバイダー
- `using`とプロジェクト設定
- プロダクションコード

既存テストコードは、設計根拠の確認と実行に限って読み取る。コメントを追加しただけのケースは、実装済みテストやSkippedテストとして数えない。

## コメントに実装可能な情報を残す

コードを書かないことと、曖昧なメモで済ませることは同じではない。実装者が後から判断をやり直さなくてよいように、テスト名、前提、操作、期待結果、テスト層、優先度をコメントへ残す。

```csharp
// TODO(test): PostImages_WhenRequestIsValid_ReturnsPng
// Given: a valid image request
// When: the client posts the request
// Then: the API returns PNG content with the documented metadata
// Level: ASP.NET Core integration
// Priority: High
```

仕様が未確定なら、期待結果を作らずブロッカーを明記する。

```csharp
// TODO(test): PostImages_WhenUpstreamTimesOut_ReturnsGatewayTimeout
// Blocked by: define the upstream timeout contract
// Expected: return the documented timeout response without internal details
// Priority: High
```

この形式なら、テスト対象のWhatは残しつつ、実装のHowを先回りしない。

## Skillの発火条件にも制約を書く

Skillは`SKILL.md`本文を読む前に、YAML frontmatterの`description`で利用可否を判断する。そのため、コメント専用であることを本文だけでなくdescriptionにも含めた。

```yaml
---
name: xunit-aspnet-core-test-case-design
description: Design behavior-focused, comment-only xUnit test plans for ASP.NET Core applications without writing test code.
---
```

UI向けの`agents/openai.yaml`も同じ契約へ揃える。既定プロンプトが単に「xUnitテストを設計する」と書かれていると、利用者の意図より広い生成を誘発するためである。

## テスト層はコードを書かなくても決められる

コメントだけの計画でも、最小のテスト層は選定できる。

| 層 | コメントで記録する境界 | 代表的な対象 |
| --- | --- | --- |
| Unit | ホストやI/Oを使わない振る舞い | Value Object、ドメイン不変条件、純粋な変換 |
| Integration | ASP.NET CoreホストとHTTP契約 | Routing、model binding、認証、ProblemDetails |
| Contract | 外部HTTPとの互換性 | status、header、JSON schema、timeout mapping |
| E2E | 主要な利用者フロー | 認証から重要操作までの横断経路 |

Unitで十分な分岐をIntegrationやE2Eへ重複させない。一方、HTTP status、serialization、DI、middleware、認証境界は、Unitのmockだけで証明したことにしない。

## 検証方法

変更後は、Skillの形式と禁止したテスト骨格の不在を確認した。

```powershell
$env:PYTHONUTF8 = '1'
python "$HOME\.codex\skills\.system\skill-creator\scripts\quick_validate.py" `
  "$HOME\.agents\skills\xunit-aspnet-core-test-case-design"
```

結果は`Skill is valid!`だった。さらに、属性直後のテストメソッドなど、実行可能なxUnit骨格を正規表現で検索し、該当なしを確認した。

変更は次のコミットに記録した。

```text
f85d929 fix: require comment-only xUnit plans
```

## 制約と再考条件

コメントはテストランナーに認識されないため、失敗もSkippedも報告されない。必ず「未実装テスト一覧」として別途管理する必要がある。

将来、利用者が実行可能なテストコードを明示的に求めた場合は、コメント専用Skillの制約をその場で無視するのではなく、別Skillへ責務を分ける方が安全である。発火条件と出力契約を分離すれば、「設計だけ」と「実装まで」の境界をレビューしやすくなる。

確認日: 2026年8月11日

## まとめ

確認日: 2026年8月11日
