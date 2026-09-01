# zenn-articles

[Zenn](https://zenn.dev/) の記事置き場。GitHub連携で、このリポジトリの
`main` に push すると自動で同期される。

## 書き方

    articles/<slug>.md

`slug` は `a-z0-9` とハイフン・アンダースコアで12〜50字。
frontmatter は次のとおり（Zenn CLI ガイドに準拠）。

```yaml
---
title: "記事のタイトル"
emoji: "🚌"        # 1文字だけ
type: "tech"       # tech か idea
topics: ["gtfs", "個人開発"]
published: false   # true にすると公開
---
```

**`published: false` のまま push すれば下書き。** Zenn上で見た目を確認して
から `true` に変えて再度 push する。

## 注意

- 記事を消すときは **Zennのダッシュボードから消す**。ファイルを消しただけ
  だと、次のデプロイで復活する（連携の仕様）。
- 連携できるリポジトリは最大2つ。
