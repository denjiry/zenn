---
title: "入力単語のサジェスト処理をRustで書いてReactから呼び出す"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Rust, webassembly, wasmpack, React]
published: false
---

これは株式会社LabBase テックカレンダー Advent Calendar 2022、15日目の記事です。

# 概要 #

編集距離を元に、フォームに入力された単語と似てる単語を辞書から引っ張ってくる処理をRustで実装し、それをwasmpackでパッケージにしてReactから呼び出すコードを書きました。

## crate-typeでrlibも指定しないと、tests以下で本体が見えない ##

crate-typeでrlibも指定しないと、tests/以下などの他のディレクトリから参照することができず、以下のようなエラーが出る。（なおrust-analyzerの定義ジャンプは動いてしまう）

```shell
error[E0432]: unresolved import `suggespell`
 --> tests/web.rs:1:5
  |
1 | use suggespell::{initialize_sub_string_dictionary, search};
  |     ^^^^^^^^^^ use of undeclared crate or module `suggespell`
```

wasm-bindgenを使うと`crate-type = ["cdylib"]`と指定することを求められるが、[wasm-packのチュートリアル](https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/template-deep-dive/cargo-toml.html#1-crate-type)によると、`wasm-pack test`のビルドをするためには`"rlib"`も指定する必要がある。

したがって、以下のように両方指定することで`wasm-pack test --headless --firefox`が動く。
``` toml
[lib]
# rlibもcrate-typeに加えないとtests/以下からインポートできない
crate-type = ["cdylib", "rlib"]
```

# まとめ #

やり残しとしては以下が思いついた

  * 辞書のビルドをパッケージのビルド時などに逃がすことで、初期化にかかる６秒くらいを削減したい
  * 今回はやってみたかったのでRustを使ったが、ピュアにTypeScriptで実装しても性能出たかどうかは気になる

# 参考 #

https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/template-deep-dive/cargo-toml.html#1-crate-type
