# byte[]を持つC# recordに内容等価性と安定hashを選んだ理由

## 結論

画像バイナリを値として表す`GeneratedImageData`では、`byte[]`の参照ではなく内容を比較する等価性を採用した。一方、mutableな配列内容はhash codeから除外し、生成後に配列が変更されてもhashが変わらない設計にした。

## 背景

Portの成功値は、外部画像生成機能から受け取った画像バイトとmedia typeをrecordで表していた。位置recordの既定実装では、メンバーごとの等価性が自動生成される。

しかし.NETの配列は既定で参照等価性を使う。同じ画像バイトを別々にmaterializeした2つの`byte[]`は、内容が同じでもrecord全体では不等価になった。

## 解決したい課題

配列の割り当て元ではなく、画像のバイト内容とmedia typeでPort値の同一性を決める。また、等価性を変更しても「等価な値は同じhash codeを返す」という契約と、hashの安定性を維持する。

## 前提・制約

- 技術的制約: .NET 8の`byte[]`は参照等価性を使用する。
- 技術的制約: `byte[]`は生成後も変更できる。
- 設計上の制約: `GeneratedImageData`は配列インスタンスではなく画像データを表す値である。
- 互換性上の制約: 既存の`GeneratedImage`もbyte sequenceによる等価性と、バイトを除外したhashを使用している。

## 検討した選択肢

### Option A: recordの既定等価性を使う

位置recordが生成する`Equals`と`GetHashCode`をそのまま使う。

#### メリット

追加実装が不要で、比較時にバイト列を走査しない。

#### デメリット

同じ内容でも別配列なら不等価になり、値の意味より割り当て方に結果が依存する。

### Option B: 内容等価性を定義し、バイトをhashから除外する

`SequenceEqual`でbyte sequenceを比較し、hash codeはmedia typeから作る。

#### メリット

別々に作られた同じ画像データを同じ値として比較できる。配列が変更されてもhash codeは変化しない。

#### デメリット

等価性比較はバイト数に比例する。異なる画像でもmedia typeが同じなら同じhash codeになり、hash collisionが増える。

## 評価軸

| 評価軸 | Option A | Option B |
| --- | --- | --- |
| 値の意味との一致 | 配列の参照に依存する | バイト内容で比較できる |
| hashの安定性 | 配列参照のhashは安定するが内容等価でない | mutableな内容を除外して安定する |
| 比較コスト | 参照比較 | byte sequenceの走査 |
| 既存型との一貫性 | `GeneratedImage`と異なる | `GeneratedImage`と同じ方針 |

## 最終判断

Option Bを採用した。`Equals`ではバイト内容とmedia typeを比較し、`GetHashCode`にはmedia typeだけを使用する。

## なぜこの選択をしたか

Port境界では、HTTP responseなどから同じ画像内容が別の配列として生成され得る。配列参照を値の一部にすると、この実装上の割り当て差がDomainへ漏れる。

一方、mutableなバイト内容をhashへ含めると、hash-based collectionへ格納した後の変更でhashが変わり、格納済みの値を検索できなくなる可能性がある。内容比較とhash安定性を別々に扱い、同じmedia typeによるcollision増加を受け入れた。

## 実装後の結果

別々の配列から作成した次の2値が等価になり、hash codeも一致することをSmall testで確認した。

```csharp
var first = new GeneratedImageData([0x89, 0x50, 0x4E, 0x47], "image/png");
var second = new GeneratedImageData([0x89, 0x50, 0x4E, 0x47], "image/png");

Assert.Equal(first, second);
Assert.Equal(first.GetHashCode(), second.GetHashCode());
```

全体のRelease testは81件成功、2件Skip、失敗0件だった。

## トレードオフ・今後の懸念

等価性比較は画像サイズに比例し、media typeだけのhashは分散が弱い。また、この判断は配列をimmutableにはしない。内容の変更自体を禁止する必要が生じた場合は、immutableな所有方法を別の判断として検討する。

## まとめ

`byte[]`を持つrecordでは、recordという宣言だけで期待する値等価性が得られるとは限らない。型が配列インスタンスではなく配列内容を意味するなら、内容比較を明示し、mutableな内容とhashの安定性を別に検討する必要がある。この判断はADR-0027へ記録した。
