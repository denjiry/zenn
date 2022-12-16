---
title: "入力単語のサジェスト処理をRustで書いてReactから呼び出す"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Rust, webassembly, wasmpack, React]
published: false
---

これは株式会社LabBase テックカレンダー Advent Calendar 2022、15日目の記事です。

# 概要 #

編集距離を元に、フォームに入力された単語と似てる単語を辞書からサジェストさせる処理をRustで実装し、それをwasm-packでパッケージにしてReactから呼び出すコードを書きました。
辞書の項目数として約12万件（JSTシソーラス）と100万件（Wikipediaのタイトル）を考慮し、十分高速に入力に対するサジェスト結果を表示させることができました。

# 背景 #

フォームへ専門用語などの難しい単語を入力しようとするとき、補完があることでユーザー体験向上や、入力ミスの削減できることがあります。
今回は問題を単純化し、特定の辞書ファイル（専門用語が１行ずつ入っているテキストファイル）から、入力と比較して編集距離が１以内の単語を候補として表示させることを目標にしました。

# 実装 #

一番外側のUIは、入力フォーム１つとその下にリストでサジェスト結果を並べる単純な実装です。実際に使用するときには候補を選んだら入力される仕組みもあったほうがよさそうですね。

``` tsx:入力欄の実装
const Contents: React.FC = () => {
  const [text, setText] = useState("");
  const { suggested } = useSuggest(text); // サジェストするカスタムhook
  const suggestedItems = suggested.map((item, index) => <li key={index}>{item}</li>);
  const handleTextChange = (e: {
    target: { value: SetStateAction<string> };
  }) => {
    setText(e.target.value);
  };
  return (
    <>
      <input value={text} onChange={handleTextChange} />
      {suggestedItems}
    </>
  );
};
```

ここで`useSaggest()`は、フォームに入力された`word`に対して候補の配列を返すカスタムhookなので、のシグネチャを`function useSuggest(word: string): { suggested: string[] } {}`と決めます。



`levenshtein`crateの、`levenshtein(a: &str, b: &str) -> usize`を使用しました。











``` typescript:サジェストを担うカスタムhook
export function useSuggest(word: string): { suggested: string[] } {
  const [suggested, setSuggested] = useState<string[]>([]);
  const wasmRef = useRef<InitOutput | null>(null);
  const [isInitialized, setIsInitialized] = useState<boolean>(false);
  useEffect(() => {
    if (!isInitialized) {
      (async () => {
        const [dictionary, wasmInstance] = await Promise.all([
          fetchDict(),
          init(),
        ]);
        wasmRef.current = wasmInstance;
        console.log("init");
        initialize_sub_string_dictionary(dictionary, 1);
        setIsInitialized(true);
      })();
    } else {
      console.log("search");
      const output: string[] = search(word);
      setSuggested(output);
    }
    return () => {
      wasmRef.current = null;
    };
  }, [isInitialized, word]);
  return { suggested };
}
```

## crate-typeでrlibも指定しないと、tests以下で本体が見えない ##

crate-typeでrlibも指定しないと、tests/以下などの他のディレクトリから参照することができず、以下のようなエラーが出てしまいます。（なおrust-analyzerの定義ジャンプは動いてしまう）

```shell
error[E0432]: unresolved import `suggespell`
 --> tests/web.rs:1:5
  |
1 | use suggespell::{initialize_sub_string_dictionary, search};
  |     ^^^^^^^^^^ use of undeclared crate or module `suggespell`
```

wasm-bindgenを使うと`crate-type = ["cdylib"]`と指定することを求められますが、[wasm-packのチュートリアル](https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/template-deep-dive/cargo-toml.html#1-crate-type)によると、`wasm-pack test`のビルドをするためには`"rlib"`も指定する必要があります。

したがって、以下のように両方指定することで`wasm-pack test --headless --firefox`が動きます。
``` toml
[lib]
# rlibもcrate-typeに加えないとtests/以下からインポートできない
crate-type = ["cdylib", "rlib"]
```

# まとめ #

やり残しとしては以下を思いつきました

  * 辞書のビルドをパッケージのビルド時などに逃がすことで、初期化にかかる６秒くらいを削減したい
  * 今回はやってみたかったのでRustを使ったが、ピュアにTypeScriptで実装しても性能出たかどうかは気になる

# 参考 #

https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/template-deep-dive/cargo-toml.html#1-crate-type
