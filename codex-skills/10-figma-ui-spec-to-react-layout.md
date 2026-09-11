# Figmaの画面仕様をReactのレイアウトへ落とし込む

## 対象読者

Figmaで定義された画面を、既存のReactコンポーネントとi18nを使って実装する開発者。

## スコープ

今回のMojica画像生成画面で確認した、画面構造・レスポンシブ幅・表示文言の反映を扱う。Figmaのデザインを自動的にReactコードへ変換する方法や、配色の完全な再現は扱わない。

## 背景

画像生成フォーム自体は実装済みだったが、Figmaにある画面の導入文、入力補助文、自動ダウンロード案内、ヘッダー・フッターを画面全体へ組み合わせる必要があった。Figmaのメタデータでは、デスクトップのフォーム幅は620px、モバイルでは左右16pxの余白を持つ358px幅として定義されている。

## 方針

画面固有の構造は`App`側で組み立て、フォームは入力と送信を担当する責務を保つ。表示文言はコンポーネントへ直接書かず、現在のロケールに対応するメッセージ定義から取得する。

```tsx
const ImageGenerationScreen = () => {
  const { locale } = useI18n();
  const messages = imageGenerationScreenMessages[locale];

  return (
    <main className={"min-w-0 flex-1 px-4 pt-8 md:px-6 md:pt-12"}>
      <div className={"mx-auto flex w-full max-w-[620px] flex-col gap-7"}>
        <section className={"flex flex-col items-center gap-3 text-center"}>
          <h1>{messages.heading}</h1>
          <p>{messages.description}</p>
        </section>
        <ImageGenerationForm locale={locale} />
      </div>
    </main>
  );
};
```

## 実装上のポイント

- `max-w-[620px]`でフォームの最大幅を固定し、`w-full`で狭い画面では縮小させる。
- `mx-auto`でデスクトップとタブレットのフォームを中央寄せにする。
- モバイル用の固定最小幅を設けず、viewport幅に追従させる。
- 日本語と英語の見出し、説明、入力補助文を同じメッセージ構造で管理する。
- `TextField`の`helperText`のような再利用可能な表示責務は共有コンポーネントへ追加する。

## 検証

| コマンド | 結果 |
| --- | --- |
| `bun run test` | 21ファイル、103テスト成功 |
| `bun run typecheck` | 成功 |
| `bun run lint` | 成功 |
| `bun run build-storybook` | 成功 |

## 事実・判断・未確認事項

- FACT: Figmaメタデータには、フォーム幅620px、モバイルの左右余白16px、導入文と入力補助文が定義されている。
- FACT: 実装では`imageGenerationScreenMessages`からロケール別の文言を取得している。
- INFERENCE: 画面構造とフォーム責務を分離すると、フォーム単体の再利用と画面レイアウトの調整を独立して行いやすい。
- 未確認事項: Figma上の全ピクセル値との自動比較やVisual Regressionは今回実行していない。

## まとめ

Figmaとの差分を修正するときは、画面構造・共有コンポーネント・i18nの責務へ分解して実装する。最大幅と可変幅を組み合わせることで、同じ構造をデスクトップからモバイルまで適用できる。

## 参考

- `frontend/src/app/views/App.tsx`
- `frontend/src/features/image-generation/components/ImageGenerationForm/ImageGenerationForm.tsx`
- `frontend/src/components/TextField/TextField.tsx`
- `frontend/src/i18n/messages.ts`
- Figma file `s5BUjZf77T4dIqvEanOTIx` のメタデータ（2026-09-04確認）
