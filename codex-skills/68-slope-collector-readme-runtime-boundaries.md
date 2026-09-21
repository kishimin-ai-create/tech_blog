---
title: "FastAPIプロジェクトのREADMEでAPIと運用境界を分けて説明する"
tags: [Python, FastAPI, README, 運用]
---

# FastAPIプロジェクトのREADMEでAPIと運用境界を分けて説明する

READMEに起動コマンドやAPI一覧を追加するとき、実装上の利用条件と、まだデプロイ環境でしか確認できない運用事項を同じ成功例として書くと、読者がローカルで証明できる範囲を誤解します。本記事では、slope-collectorのREADME更新を題材に、実ファイルから確認できるAPI契約と、Linux環境へ持ち越す検証境界を分けて記載した方法を整理します。

## 結論

- 開発APIと本番運用の境界をREADMEで明示すると、ローカル確認の範囲を過大に見せずに済む。
- APIエンドポイント、環境変数、Composeのポート、収集コマンドは、実装・Compose・CIに存在する値だけを記載する。
- systemd timerの実起動やjournaldの確認を未実施なら、READMEでも「デプロイ環境が必要な確認」として残す。

## 変更前の課題

変更前のREADMEは、プロジェクトの説明としては短く、初めて読む開発者が次の情報をリポジトリ内から探す必要がありました。

- Python、uv、FastAPI、MySQLの前提バージョン
- `.env.example`を使ったローカル設定の作り方
- APIの起動方法とComposeのポート
- レコード取得APIのクエリパラメータ
- Alembicと収集CLIの実行方法
- Windows開発PCだけでは確認できないsystemdの範囲

READMEの不足を埋めるために、値を推測して追記するのではなく、`pyproject.toml`、`Dockerfile`、`compose.yaml`、APIルーター、CIワークフロー、systemd unitを照合しました。

## APIの利用条件を実装に合わせる

READMEには、実装で確認できたエンドポイントだけを載せました。

| Method | Path | Scope |
| --- | --- | --- |
| `GET` | `/health` | 開発APIのヘルス確認 |
| `GET` | `/sources` | 開発環境でのソース一覧 |
| `GET` | `/entities` | エンティティ一覧と`source_id`絞り込み |
| `GET` | `/records` | 日付・エンティティ・ページング付きのレコード一覧 |
| `GET` | `/records/{record_id}` | レコード本文とアセットの取得 |

`/sources`のような開発用ルートは、すべての環境で利用できるように見せないことが重要です。READMEでは開発APIの説明として記載し、本番環境のルーター構成まで拡張して断定しないようにしました。

## 設定例と実行コマンドを追跡可能にする

追跡対象の環境ファイルはプレースホルダーに限定し、実値を含む`.env`はREADMEにも転載しません。ローカル起動手順は、リポジトリに存在する例示ファイルとコマンドへ接続します。

```powershell
Copy-Item .env.example .env
Copy-Item .env.development.example .env.development
uv sync --all-groups
uv run uvicorn app.main:app --reload --port 8080
```

Compose利用時のAPIバインドは`127.0.0.1:8080`、MySQLは`127.0.0.1:3307`です。READMEでは、これらを`compose.yaml`から確認した値として示しました。データベースの接続文字列は`mysql+pymysql`ドライバーを要求するため、`DATABASE_URL`の失敗を実際に遭遇したトラブルシュートとして分離しています。

## ローカル検証とデプロイ検証を分ける

このリポジトリでは、Windows開発PCでAPI、設定、Docker、CI上のsystemd unit構文を確認できます。一方、timerの実起動とjournaldログの確認は、実Linuxのデプロイ環境が必要です。

READMEには、次のように検証済みと未確認を分けて記載しました。

- FACT: `GET /health`、Compose設定、Alembic、収集CLI、Linux CIのunit構文検証が確認対象にある。
- FACT: 実Linux環境でのtimer実行とjournald確認は未実施である。
- INFERENCE: READMEでこの境界を明示すると、ローカル検証済みであることを本番運用済みと誤読しにくい。
- PROPOSAL: ステージング相当のLinux環境でtimerを一度実行し、service終了状態とjournaldを確認する。

未確認の運用手順を成功結果として書かないことが、READMEの信頼性を保つための条件になります。

## 検証

README変更後に実行した確認は、文章とリポジトリの整合性に限定しました。

```powershell
git diff --check
```

結果は成功でした。さらに、目次リンク、各セクションのトップリンク、READMEが参照する設定ファイルとディレクトリの存在をスクリプトで確認しました。コードの振る舞いを変更していないため、今回のREADME変更ではアプリケーションテストを再実行していません。

READMEの変更はコミット`27d72f4`（`docs: document project usage and operations`）に含まれます。その後、ローカル`main`を`origin/main`へfast-forwardし、`docs/update-readme`を最新の`main`から作成しました。ブランチ間の差分がないため、このブランチからPull Requestは作成していません。

## 学びと制約

READMEの価値は情報量ではなく、読者がその記述をどのファイルとコマンドで再確認できるかにあります。APIの公開範囲、環境変数の境界、Composeのポート、systemdの未確認事項を分けることで、ローカル開発の手順とデプロイ前の残作業を同じ文書で追跡できます。

ただし、READMEは実Linux環境でのtimer実行を代替しません。また、実際の取得対象、セレクター、秘密情報、収集データを記載していないため、外部取得の実運用結果を示す記事でもありません。

## 参考

- `slope-collector/README.md`
- `slope-collector/pyproject.toml`
- `slope-collector/Dockerfile`
- `slope-collector/compose.yaml`
- `slope-collector/app/api`
- `slope-collector/deploy/systemd`
- `slope-collector` commit `27d72f4`

確認日: 2026-09-21
