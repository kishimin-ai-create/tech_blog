# CodexとClaude CodeのセッションJSONLを同じ修復フローで扱う

## はじめに

CodexとClaude Codeは、どちらもローカルにJSONL形式のセッション履歴を持つが、レコード構造は同じではない。失敗セッションからSkill改善の根拠を抽出するには、形式差を吸収しつつ、内部のreasoningやthinkingを成果物へ持ち出さない境界が必要になる。

この記事では、2026年9月18日にローカルで観測した両形式を、共通の可視イベント列へ変換した実装と検証をまとめる。実際の会話本文や認証情報は掲載しない。

## 前提・環境

- OS: Windows 11
- Runtime: Python 3.12.4
- 対象: CodexとClaude CodeのローカルJSONL
- 実装先: `$HOME/.agents/skills/repair-session/scripts/repair.py`
- 検証日: 2026年9月18日

セッション形式は公開APIとして固定されているとは限らない。この記事は観測時点の構造とテストfixtureに基づく。

## やってみた結果

セッションUUIDまたはJSONLパスを1つ受け取り、次の共通イベントへ変換できるようにした。

- ユーザーの可視メッセージ
- アシスタントの可視メッセージ
- ツール呼び出し
- ツール結果
- 欠損または不正レコードの位置

CodexのreasoningとClaude Codeのthinkingは抽出しない。ケース保存先は、Codexなら`$HOME/.codex/agent-repair/`、Claude Codeなら`$HOME/.claude/agent-repair/`へ分離した。

## 実装・検証

### Step 1：保存場所とレコードから形式を判定する

UUIDが渡された場合、Codexでは`sessions`と`archived_sessions`、Claude Codeでは`projects`配下のJSONLを探索する。候補が1件でなければ自動選択しない。

読み込んだレコードでは、次のキーで形式を判定した。

```python
def identify_tool(records):
    if any(record.get("type") in {"session_meta", "response_item"}
           for _, record in records):
        return "codex"
    if any(record.get("type") in {"user", "assistant"}
           and "sessionId" in record for _, record in records):
        return "claude"
    raise ValueError("Session format is not recognized")
```

ファイルの配置だけで決めず、配置から期待した形式と内容から判定した形式が一致することも確認する。これにより、誤ったホーム配下へ置かれたJSONLをそのまま処理しない。

### Step 2：Codexの可視イベントを抽出する

Codexでは、`response_item`のうち`message`とツール関連イベントを扱う。`session_meta`と`turn_context`からはセッションID、作業ディレクトリ、Automation IDなどのメタデータを取得する。

`reasoning`はイベントへ追加しない。壊れたJSON行やcompactionは、会話が完全ではないことを示す`gaps`として記録する。

### Step 3：Claude Codeのcontent blockを展開する

Claude Codeでは、トップレベルの`user`または`assistant`レコード内に`message`があり、`message.content`は文字列またはblockの配列だった。

配列では、`text`を可視メッセージ、`tool_use`をツール呼び出し、`tool_result`を結果として変換する。`thinking` blockは読み飛ばす。

```python
if item_type == "tool_use":
    events.append({"kind": "tool_use", "id": item.get("id"),
                   "name": item.get("name"), "input": item.get("input")})
elif item_type == "tool_result":
    events.append({"kind": "tool_result",
                   "tool_use_id": item.get("tool_use_id"),
                   "content": item.get("content")})
```

ツール結果には秘密情報が含まれる可能性があるため、ケースはリポジトリ外へ置く。修復適用後は会話本文とサンプル出力を削除する。

### Step 4：形式差をfixtureで固定する

実際のセッションをテストへコピーせず、必要なキーだけを持つ最小fixtureを作った。テストでは次を確認した。

- Codexの可視メッセージを抽出し、reasoningを含めない。
- Claude Codeのtext、tool use、tool resultを抽出し、thinkingを含めない。
- ツールごとのケース保存先を選ぶ。
- CLIへ両方のホームパスを明示できる。
- 資格情報ファイル、パス脱出、後続編集を拒否する。
- 部分失敗時に先行書き込みを復元する。

実行コマンドは次のとおりである。

```powershell
$env:PYTHONDONTWRITEBYTECODE = '1'
python -m coverage run --branch `
  --source=skills/repair-session/scripts `
  -m unittest discover -s skills/repair-session/tests -v
python -m coverage report -m
```

8件のテストが成功し、statement・branch込みカバレッジは85%だった。

## つまずいたところ

### 問題：同じ「会話履歴」でも構造が異なる

Codexのイベントは`type: response_item`の`payload`に入り、Claude Codeではトップレベルのレコードと`message.content` blockに分かれていた。単一の汎用JSON探索で本文を集めると、thinkingや無関係なメタデータまで混ざる可能性がある。

### 原因

2つのツールでイベントモデルと保存場所が異なるためである。また、同じ`content`でも文字列、text block、tool resultなど意味が異なる。

### 解決

形式判定と形式別parserを分け、最後に共通イベントへ変換した。未知の形式を推測で処理せず、認識できない場合は停止するようにした。

## 学んだこと

異なるエージェントのセッション履歴を統合するときは、先に共通スキーマを作るより、各形式で「公開してよい可視イベント」を明示的に列挙する方が境界を確認しやすい。

また、抽出成功だけでは十分ではない。履歴の欠損、形式の不一致、IDの重複をエラーまたは`gaps`として残すことで、診断根拠が不完全なときに断定を避けられる。

## まとめ

CodexとClaude CodeのJSONLは、保存場所もレコード構造も異なる。形式ごとにparserを分け、reasoning／thinkingを除外し、可視メッセージとツールイベントだけを共通化することで、同じ修復ワークフローへ載せられた。

この実装は観測時点の形式に依存する。ツールの更新後は、実セッションのキー構造を内容非表示で確認し、失敗するfixtureを先に追加してからparserを更新する。

## 参考資料

- [tankadoko/agent-repair](https://github.com/tankadoko/agent-repair)
