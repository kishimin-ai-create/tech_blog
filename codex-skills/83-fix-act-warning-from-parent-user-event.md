# `user.type`の`act`警告を依存パッケージの解決先から直す

## 結論

`await user.type(...)`の実行時にReactの`act`警告が出た原因は、プロジェクトに`@testing-library/user-event`が直接依存として登録されず、Node.jsが親フォルダーにあった別のインストールを解決していたことだった。user-eventをプロジェクトの開発依存に追加すると、React Testing Libraryと同じ`@testing-library/dom`を共有し、警告が消えた。

## 発生した問題

MUIの`TextField`へ文字入力するテストで、`await user.type(...)`を追加すると次の警告が出た。

```text
An update to ForwardRef(FormControl) inside a test was not wrapped in act(...).
An update to SampleForm1 inside a test was not wrapped in act(...).
```

テストではReact Testing Libraryの`render`とuser-eventの`user.type`を併用していた。

## 調査

プロジェクト内にuser-eventがあるか確認した。

```powershell
npm ls @testing-library/user-event --all
```

結果は`(empty)`だった。一方、Node.jsのモジュール解決は`C:\Users\Kazum\node_modules\@testing-library\user-event`を返し、親フォルダーにあるuser-event 14.6.1が使われていた。

警告のスタックには、そのuser-event側から`C:\Users\Kazum\node_modules\@testing-library\dom`を読み込んだ経路が現れていた。React Testing Library 16.3.3はプロジェクト内の別の`@testing-library/dom`を使っていた。

## 原因

React Testing Libraryは自身が読み込んだDOM Testing Libraryの設定にReactの`act`ラッパーを登録する。user-eventが別のDOM Testing Libraryインスタンスを使うと、その設定を共有できない。

したがって、`await`が警告の原因ではなかった。非同期入力の各イベントが、React Testing Library側で設定された`act`ラッパーを通らない依存構成が原因だった。

## 解決方法

user-eventをプロジェクトの開発依存に追加した。

```powershell
npm install --save-dev @testing-library/user-event
```

追加後は`@testing-library/user-event` 14.6.7と、React Testing Libraryおよびuser-eventが共有する`@testing-library/dom` 10.4.2がプロジェクト内で解決された。

テストでは操作を最後まで待ち、その直後に値を検証する。

```tsx
await user.type(input, "のんびり");
expect(input).toHaveValue("のんびり");
```

`await user.type`後の更新確認に追加の`waitFor`は必要なかった。

## 動作確認

- 修正前: `node .\node_modules\jest\bin\jest.js --runInBand src/field.test.tsx`でテストは通ったが、`FormControl`とコンポーネントの`act`警告が出た。
- 修正後: 同じコマンドで2テストが通り、警告は出なかった。
- `npm test`: 2スイート、3テスト成功。
- `npm run build`: TypeScriptチェックとViteビルド成功。
- `npm run lint`: 成功。

## 事実・判断・未確認事項

- FACT: 修正前のプロジェクト依存一覧にuser-eventはなく、Node.jsは親フォルダー内のuser-event 14.6.1を解決した。
- FACT: 修正後はプロジェクト内のuser-event 14.6.7とDOM Testing Library 10.4.2が解決され、同じテストで警告が出なかった。
- INFERENCE: 別々のDOM Testing Library設定インスタンスが警告の直接原因だった。スタックトレースのモジュールパスと、依存追加前後の挙動を根拠にした。
- 未確認: すべてのReactやTesting Libraryのバージョン組み合わせで同じ症状が起きるかは検証していない。

## まとめ

`act`警告が出たら、テストコードに`act`を足す前に依存の解決先を調べる。テストランナーとTesting Libraryの依存をプロジェクトへ明示すれば、別プロジェクトや親フォルダーのパッケージに偶然依存せずに済む。

確認日: 2026年9月28日
