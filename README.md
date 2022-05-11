# ⚓ The Anchor Book中文版

快速上手[Anchor](https://anchor-lang.com), 为搭建安全, 可靠的Solana apps而生的框架。

目前更新至[英文版](https://github.com/project-serum/anchor-book)的这个[commit](https://github.com/project-serum/anchor-book/tree/0099a11a65390e16b6dd6e9b192758c14ffeeb72)。


## 🤝 贡献

尽管提issue或者PR。


## Programs示例代码

你可以在[programs directory](./programs/)找到书中使用的示例代码。


## 💻 在本地运行The Anchor Book

在Mac运行, 如果你还没装，需要安装[Homebrew](https://brew.sh/)

然后，运行下面的命令:

```sh
brew upgrade
brew install mdbook
```

然后, clone这个repo并且运行书的本地服务:

```sh
git clone https://github.com/project-serum/anchor-book-cn.git
cd anchor-book-cn
mdbook serve
```
书就可以在浏览器的`http://localhost:3000`访问了。


### 运行老的版本（需要到[英文版](https://github.com/project-serum/anchor-book)）

如果你想了解之前版本的anchor的某些功能，你可以check out对应的分支，比如，`v0.21.0`有所有书中所有在`v0.22.0`发布之前的内容。


## License

The Anchor Book is licensed under [Apache 2.0](./LICENSE).

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in Anchor by you, as defined in the Apache-2.0 license, shall be
licensed as above, without any additional terms or conditions.
