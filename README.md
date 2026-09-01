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

## 日本語で書くときの落とし穴

**強調の `**` が、約物の直後では閉じられない。**

```
✗  **「これから使う日を覆っているか」**でした。
✓  「**これから使う日を覆っているか**」でした。
```

CommonMarkの規則で、閉じる `**` の直前が約物（`」`『。』`）` など）で、
直後が普通の文字だと強調として成立しない。日本語は単語を空白で区切らない
ため、英語では起きない失敗が出る。**記号がそのまま画面に出る。**

書いたら必ず `npx zenn-cli@latest preview` で描画を確かめること。
記法の誤りは、原稿を読んでいるだけでは分からない。

確認のしかた（描画後にブラウザのコンソールで）:

```js
const t = document.querySelector('.znc').innerText;
(t.match(/\*\*/g) || []).length   // 0 でなければ変換されていない箇所がある
```

