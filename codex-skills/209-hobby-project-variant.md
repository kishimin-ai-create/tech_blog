# 共通の作品データに Hobby 専用カードを用意する

## 背景

profile-hub は Engineering と Hobby を別の公開体験として扱う。作品データは共通の Portfolio API からセクション別に取得するが、Hobby の作品一覧は見た目が未整備のままだった。

## デザインと実装

既存の Hobby のお気に入りカードと Engineering の Portfolio カードを参照し、Hobby の作品カードをデスクトップ・モバイル向けの編集可能な Figma 下書きとして作成した。タイトル、説明、技術、任意のリンクをカード内に並べ、Hobby の文字サイズと角丸を使う。

実装ではデータの取得やリンク処理を共有したまま、Hobby に専用のスタイルを与えた。リポジトリやデモがない場合の表示契約は維持する。作成した Figma 案はレビュー前の下書きであり、承認済みデザインではない。

## 検証と限界

長い日本語を含むカードについて、375px と 1280px の Chromium・WebKit でカードが画面幅を超えず、リンクが表示されることを確認した。公開画面の57テスト、通常ブラウザーテスト32件、整形・型チェック・Lint・HTML 検査・ビルドが成功した。初回のスクリーンショットでは作品 API 以外が未固定でお気に入りがエラーになったため、関連 API も固定して Chromium の両幅を撮影し直した。

根拠: [Hobby 作品カード PR #221](https://github.com/kishimin-ai-create/profile-hub/pull/221)、[デスクトップ案](https://www.figma.com/design/cl17hpDO7KrU9S6WYPvILk/profile-hub?node-id=147-2)、[モバイル案](https://www.figma.com/design/cl17hpDO7KrU9S6WYPvILk/profile-hub?node-id=147-7)。PR とデザインはレビュー待ちで、本番配備は行っていない。
