# Skillを別のPCへ移植するためのパス監査を作る

## はじめに

Skillを複数のPCで再利用するには、手順だけでなく入力先・出力先・ファイル名に含まれる環境依存の値も確認する必要があります。本記事では、既存Skillを一括変換せず、移植性に影響するパスを検出し、共有用レポートで機械固有の値を伏せる仕組みを紹介します。

## 前提・環境

- 対象: Git管理された共有Skill集合
- 対象ファイル: 各Skillの`SKILL.md`
- 実行: Pythonスクリプト
- 方針: 監査は読み取り専用。書き換えは対象を選んでから実施

## やってみた結果

`portable-skill-paths` Skillと`audit_skill_paths.py`を追加しました。完全なSkillルートを監査した結果、94件の`SKILL.md`から64件の参照を検出しました。内訳は`$HOME`または`$CODEX_HOME`を使うcanonicalが62件、コンテナ内で固定される`/workspace`が2件です。machine-specificとreviewは0件でした。

## 実装・検証

### パス参照を監査する

監査対象のSkillルートを引数に渡します。`--redact`を付けると、レポートのルートや検出値を共有用プレースホルダーへ置き換えます。

```powershell
python <AGENTS_SKILLS_ROOT>/portable-skill-paths/scripts/audit_skill_paths.py <AGENTS_SKILLS_ROOT> --redact
```

レポート上では、ユーザー環境に依存する値を`<HOME>`や`<PROJECT_ROOT>`として表示します。実行に使うローカルパスは変更しません。

### URLの誤検出を修正する

初回の検出では、`https://`の`s://`部分をWindowsドライブパスと誤認していました。ドライブパスの直前が英字でないことを条件に加え、URLをファイルパスとして扱わないようにしました。

Cloudflare Sandboxの例にある`/workspace`はホストPCのパスではなく、コンテナ内の固定パスです。そのため`runtime-fixed`として分類し、移植時に置換する値と区別しました。

### 検証する

```powershell
python <CODEX_HOME>/skills/.system/skill-creator/scripts/quick_validate.py <PORTABLE_SKILL_PATHS_ROOT>
python <AGENTS_SKILLS_ROOT>/portable-skill-paths/scripts/audit_skill_paths.py <AGENTS_SKILLS_ROOT> --redact
```

`quick_validate.py`は成功しました。監査は`Scanned Skill files: 94`、`Findings: 64`を返し、分類はcanonical 62、runtime-fixed 2でした。

## 学んだこと

パス監査では、絶対パスを探すだけでは不十分です。URLのような別構文を除外し、実行環境内で固定されるパスを機械固有パスと分ける必要があります。また、監査と編集を分離し、対象Skillと差分を確認してからパラメーター化する方が安全です。

## 制約と次の課題

スクリプトは正規表現による監査であり、すべてのシェル構文や独自のパス表記を意味解析するものではありません。新しいパス形式や実行環境の慣例を追加した場合は、分類条件の見直しが必要です。既存Skillの一括書き換えは行っていません。

## まとめ

Skillの移植性を確認するため、読み取り専用のパス監査と共有用の伏せ字表示を用意しました。現在の監査では、既存Skillに書き換え対象の機械固有パスは見つかりませんでした。今後はSkillを共有・移行する前に監査を実行し、新たな非canonical参照だけを個別にレビューします。

## 参考資料

- `portable-skill-paths/SKILL.md`
- `portable-skill-paths/scripts/audit_skill_paths.py`
- `b6d5446 feat: audit portable Skill paths`
- `8381613 fix: avoid URL path false positives`
