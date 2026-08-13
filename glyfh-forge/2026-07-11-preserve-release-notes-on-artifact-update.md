# Releaseの成果物更新で本文を消さない

## はじめに

GitHub Release は、成果物だけを更新したいことがある。

たとえばタグを直したあと、同じ Release に wheel や tar.gz を載せ直す場合。

このとき本文まで空で上書きされると困る。
手で書いた release notes や、自動生成された説明が消えてしまうからだ。

今回の release workflow では、これを追加した。

```yaml
omitBodyDuringUpdate: true
```

`allowUpdates: true` は既存 Release の更新を許可する設定。
`omitBodyDuringUpdate: true` は、その更新時に本文を触らないための設定。

成果物だけ更新したいなら、本文は守る。
小さい設定だけど、事故を防ぐには効く。

## まとめ

小さい設定だけど、事故を防ぐには効く。
