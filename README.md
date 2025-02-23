# Zenn CLI

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

### プレビューする

この階層で `path/to/zenn`

```sh
npx zenn preview
```

### 新規記事の作成

- slug-name は 12 文字以上
```sh
npx zenn new:article --slug <slug-name> --title "<title>" --type tech --emoji 🐔
```

### 画像を挿入する

- 画像もGithubで管理する<br> 以下のようにディレクトリを作成して絶対パスでファイルを指定するだけ
```tree
images
└── directory
    └── image1.png
```
```
![](/images/directory/image1.png)
```

### リンク

[Zenn CLIをインストールする](https://zenn.dev/zenn/articles/zenn-cli-guide)

[Zenn CLIで記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)
