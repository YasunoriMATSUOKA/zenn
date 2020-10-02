# Zenn Contents

- [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [📘 Markdown guide](https://zenn.dev/zenn/articles/markdown-guide)

## Cheatsheet

### Create New Article

- Command

```sh
npx zenn new:article --slug 記事のスラッグ --title タイトル --type idea --emoji ✨
```

- Options

```sh
---
title: "" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

ここから本文を書く
```

### Create New Book

- Command

```sh
$ npx zenn new:book
# 本のslugを指定する場合は以下のようにします。
# npx zenn new:book --slug ここにスラッグ
```

### Image Upload Link

- [📘 Image upload link](https://zenn.dev/dashboard/uploader)
