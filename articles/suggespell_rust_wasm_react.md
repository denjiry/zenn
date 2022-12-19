---
title: "入力単語のサジェスト処理をRustで書いてReactから呼び出す"
emoji: "🦀"
type: "tech"
topics: [Rust, WebAssembly, wasmpack, React]
published: true
---

これは株式会社LabBase テックカレンダー Advent Calendar 2022、15日目の記事です。
https://qiita.com/advent-calendar/2022/labbase
13日目の記事は sotabkw さんによる「Reactとアクセシビリティ(a11y)対応」という記事でした。ぜひご覧ください。
https://qiita.com/sotabkw/items/9e78bf1103117b091ee5
今回は、Rustで書いた入力単語のサジェスト処理を紹介します。

# 概要 #

編集距離を元に、フォームに入力された単語と似てる単語を辞書からサジェストさせる処理をRustで実装し、それをwasm-packでパッケージにしてReactから呼び出すコードを書きました。
辞書の項目数として約12万件（[JSTシソーラス](https://dbarchive.biosciencedbc.jp/jp/mecab/data-1.html)）と約100万件（Wikipediaのタイトル）を考慮し、十分高速に毎タイプ入力時にサジェスト結果を表示させることができました。

# 背景 #

フォームへ専門用語などの難しい単語を入力しようとするとき、補完があることでユーザー体験向上や、入力ミスを削減できることがあります。
今回は問題を単純化し、特定の辞書ファイル（専門用語のみが1行ずつ入っているテキストファイル）から、入力と比較して編集距離が1以内の単語を候補として表示させることにしました。このため、前方一致している単語のサジェストなどは次の機会にゆずることにしました。
また十分高速に動くかどうかは、フォーム内でコピペしたりUndoしたりを手動で繰り返してみることで確かめることにしました。

# 実装 #
ブラウザのフォームで文字を入力するごとにサジェストを算出して更新する処理を実装したので、UI・サジェスト処理のアルゴリズム・Wasmを用いた機能のインポートとエクスポートについて順に説明します。

## UI ##

UIは入力フォーム１つとその下にリストでサジェスト結果を並べる単純な実装で、以下のようなコードにしました。実際に使用するときには候補を選んだら入力される仕組みもあったほうがよさそうですね。

```tsx:入力欄の実装
const Contents: React.FC = () => {
  const [text, setText] = useState(""); // 入力フォームの中身
  const { suggested } = useSuggest(text); // サジェストするカスタムhook
  // サジェスト結果
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

ここで、`useSuggest()`はフォームに入力された`word`に対して候補の配列を返すカスタムhookとしたいので、`useSuggest()`のシグネチャを`function useSuggest(word: string): { suggested: string[] }`と決めます。これによって、Rust上の関数も入力は文字列一つで、返り値は文字列複数であると自然に決まるので、次は実際にサジェスト処理をするコードを書いていきます。

## サジェスト処理のアルゴリズム ##

まずWasmのインポート・エクスポートのことを考えず、サジェスト処理のアルゴリズム面について考えます。編集距離の算出には、[`levenshtein`](https://crates.io/crates/levenshtein)の、`levenshtein(a: &str, b: &str) -> usize`を使用しました。`levenshtein`は、単純に入力文字列`a`と`b`の編集距離を計算してくれる関数で、以降編集距離の算出はこの関数を使います。
あとは入力文字列と辞書の項目を`levenshtein()`で計算すればよいのですが、実は項目数が約12万件（JSTシソーラス）の場合は、単純に項目すべてで回すループ内で`levenshtein()`を計算しても十分高速に動作します。これだとあまり工夫の余地がないので、約100万件（Wikipediaのタイトル）の辞書ファイルを新たに使用することにし、単純なループで全項目を回すとフォームへの文字入力がままならなくなるほど遅くなることを確認しました。

ちょっとググってみたところ、編集距離を算出する候補自体の数をうまいこと減らすアルゴリズムが見つかったので、サジェスト処理のアルゴリズムにはこれを使用することにしました。

藤澤 信夫, 森田 和宏, 泓田 正雄, 青江 順一 , "部分文字列を用いた綴り誤りに対する訂正候補の高速検索手法", 言語処理学会第13回年次大会, 962-965, 2007年3月.
https://www.anlp.jp/proceedings/annual_meeting/2007/pdf_dir/PB3-3.pdf

ざっくり説明すると、あらかじめ辞書の各項目の文字列を`許容編集距離+1（つまり今回は2分割）`個の部分文字列に等分割して、それぞれをキーとした連想配列を構築することで、もしクエリとの編集距離が許容値以下だった場合は、連想配列のキーのどちらか片方とは必ず一致することを利用し、編集距離を算出する候補自体の数をうまいこと減らしています。この連想配列と構築する関数、実際に候補を出力する関数をセットで`SubStringDictionary`という構造体に実装した結果、以下のようなコードになりました。


```rust:substring/src/lib.rs
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct SubStringDictionary {
    distance: usize,
    dict: HashMap<String, Section>,
}

type Section = HashMap<usize, Vec<String>>;
impl SubStringDictionary {
    pub fn new(correct_strings: Vec<Vec<char>>, distance: usize) -> Self {
        let mut dict = HashMap::<String, Section>::new();
        for correct_string in correct_strings {
            let dividing_number = correct_string.len() / (distance + 1);
            let sub_correct_strings = correct_string.chunks({
                // check precondition for core::slice::chunks
                if dividing_number == 0 {
                    continue;
                }
                dividing_number
            });
            for (chunk_number, sub_correct_string) in sub_correct_strings.enumerate() {
                let scs_len = sub_correct_string.len();
                if scs_len != dividing_number {
                    continue;
                }
                let sub_correct_string: String = sub_correct_string.iter().collect();
                let section = dict
                    .entry(sub_correct_string)
                    .or_insert_with(|| Section::new());
                section
                    .entry(scs_len)
                    .or_insert_with(|| vec![])
                    .push(correct_string.iter().collect())
            }
        }
        Self { distance, dict }
    }

    pub fn search(&self, query: &[char]) -> Vec<String> {
        let mut candidates: Vec<String> = vec![];
        let dividing_number = query.len() / (self.distance + 1);
        let sub_strings = query.chunks({
            // check precondition for core::slice::chunks
            if dividing_number == 0 {
                return candidates;
            }
            dividing_number
        });
        for sub_string in sub_strings {
            if sub_string.len() != dividing_number {
                continue;
            }
            let sub_string: SubString = sub_string.iter().collect();
            let section = match self.dict.get(&sub_string) {
                Some(section) => section,
                None => continue,
            };
            for (correct_str, _num) in section.values().flatten() {
                let levenshtein =
                    levenshtein::levenshtein(&query.iter().collect::<String>(), correct_str);
                if levenshtein <= self.distance {
                    candidates.push(correct_str.to_string());
                }
            }
        }
        candidates.sort();
        candidates.dedup();
        candidates
    }
}
```

無事にアルゴリズムが出来上がったので、さらにReactから呼べるようにラップしていきます。


## RustコードのWasmエクスポート ##

いよいよWasmを使って、普通のReactからRustのコードを利用する話に入ります。
まず、[wasm-packでブラウザ用パッケージを作るチュートリアル](https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/index.html)に沿って設定を行いましょう。

https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/index.html

↑↑のアルゴリズムの実装は`substring`というcrateにわけたので、Wasmとしてエクスポートするcrate（`suggespell`）から`substring`を

```shell:Rust側のディレクトリ構成
$ exa -T --git-ignore
suggespell
  ├── Cargo.toml
  ├── pkg
  ├── src
  │  └── lib.rs
  ├── substring
  │  ├── Cargo.toml
  │  ├── README.md
  │  └── src
  │     └── lib.rs
  └── tests
     └── web.rs
```
のように配置し、`substring::SubStringDictionary`をラップした関数に

```rust
use substring::SubStringDictionary;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn initialize_sub_string_dictionary(correct_strings: &str, distance: usize) {
    SubStringDictionary::new(correct_strings, distance)
    ?????
}

#[wasm_bindgen]
pub fn search(query: &str) -> Box<[JsValue]> {
    let sub_string_dictionary = ?????
    sub_string_dictionary.search(&query)
}
```
のように、`#[wasm_bindgen]`をつけます。
しかし、構築が重いオブジェクトが無いアルゴリズムならばこれでよかったのですが、構築した`substring::SubStringDictionary`をだれがどのように持つかは自明ではないため、少し工夫をする必要があります。Wasm上で構築したでかいオブジェクトをJavaScript側で利用する方法として
1. WasmからJavaScript側に[serde-wasm-bindgen](https://crates.io/crates/serde-wasm-bindgen)を使って送信し、以降はJavaScriptで使いたいように使う
2. Wasmがオブジェクトを持ったままで、JavaScriptには操作をラップした薄い関数を公開する[^1]

の2種類を検討しましたが、今回は2つ目にしました。というのも1つ目を使うと、辞書オブジェクト本体はJavaScriptの対応するオブジェクト（Rustの`HashMap`→JavaScriptの`Map`など）によしなに変換されて送信されるのですが、型はany型になってしまうため、Rust側でオブジェクトに実装したメソッドが使用できないからです。
Wasmにオブジェクトをもたせるためには`once_cell::sync::OnceCell`を使います。`initialize_sub_string_dictionary()`では、構築した`substring::SubStringDictionary`を

```rust
static SUB_STRING_DICTIONARY: OnceCell<SubStringDictionary> = OnceCell::new();
```

と定義した`SUB_STRING_DICTIONARY`に保存し、`search()`の中で`SUB_STRING_DICTIONARY`から取り出して`sub_string_dictionary.search()`でサジェスト処理を行わせます。

以上プラス[wasm-bindgenで許されている入出力時の型](https://rustwasm.github.io/wasm-bindgen/reference/types.html)を参考によしなに型変換を追加することで、最終的にRust+Wasm側は以下のようなコードになります。


```rust:suggespell/src/lib.rs
use wasm_bindgen::prelude::*;
use once_cell::sync::OnceCell;
use substring::SubStringDictionary;

static SUB_STRING_DICTIONARY: OnceCell<SubStringDictionary> = OnceCell::new();

#[wasm_bindgen]
pub fn initialize_sub_string_dictionary(correct_strings: &str, distance: usize) {
    let correct_strings: Vec<&str> = correct_strings.split_whitespace().collect();
    let correct_strings: Vec<Vec<char>> = correct_strings
        .iter()
        .map(|s| s.chars().collect::<Vec<char>>())
        .collect();
    // check whether if SUB_STRING_DICTIONARY is already built
    if !SUB_STRING_DICTIONARY.get().is_none() {
        return;
    }
    SUB_STRING_DICTIONARY
        .set(SubStringDictionary::new(correct_strings, distance))
        .expect_throw("fail to set SUB_STRING_DICTIONARY");
}

#[wasm_bindgen]
pub fn search(query: &str) -> Box<[JsValue]> {
    let query = query.chars().collect::<Vec<_>>();
    let sub_string_dictionary = SUB_STRING_DICTIONARY
        .get()
        .expect_throw("the dictionary is not initialized");
    let suggested_items = sub_string_dictionary.search(&query);
    let suggested_items: Vec<_> = suggested_items
        .into_iter()
        .map(|v| JsValue::from_str(&v))
        .collect();
    suggested_items.into_boxed_slice()
}
```

## ReactへのWasmのインポート ##

上のコードをインポートするために、まずReactの方のpackage.jsonにビルド用のコマンドである`"build:wasm"`を追加します。

```json:package.json
...
"scripts": {
  "build": "yarn build:wasm && tsc && vite build",
  "build:wasm": "cd suggespell && wasm-pack build --target web --out-dir pkg && cd .. && yarn upgrade suggespell",
  "dev": "vite",
  ...
},
```
`wasm-pack build --target web --out-dir pkg`によって`./suggespell/pkg`にyarnでインポート可能なパッケージが用意されます。あとはこれをReact側でインポートすればよいので、普通のローカルにあるパッケージと同様に`"dependencies"`に`suggespell`を追加します。

```json:package.json
"dependencies": {
  ...
  "react": "^18.0.0",
  "suggespell": "./suggespell/pkg"
},
```
これでReact側からインポートできるようになったので、最後に`useSuggest()`を実装します。


```typescript:useSuggestの実装全体
import { useEffect, useRef, useState } from "react";
import init, {
  InitOutput,
  initialize_sub_string_dictionary,
  search,
} from "suggespell";

async function fetchDict(): Promise<string> {
  const file = await fetch("jawiki.txt");
  return file.text();
}

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

以上で実装が完了しました。動かしてみたところ、`initialize_sub_string_dictionary()`に概ね6秒かかってしまいますが、`search()`そのものはタイピング時に気にならないくらいには高速に動作しました。参考にしたアルゴリズムがうまく計算を軽くしてくれたと思われます。

# ハマったところ #

## crate-typeでrlibも指定しないと、tests以下で本体が見えない ##

crate-typeでrlibも指定しないと、tests/以下などの他のディレクトリから参照することができず、`wasm-pack test --headless --firefox`でテストを走らせると以下のようなエラーが出てしまいます。（なおrust-analyzerの定義ジャンプは動いてしまう）

```shell
error[E0432]: unresolved import `suggespell`
 --> tests/web.rs:1:5
  |
1 | use suggespell::{initialize_sub_string_dictionary, search};
  |     ^^^^^^^^^^ use of undeclared crate or module `suggespell`
```

wasm-bindgenを使うと`crate-type = ["cdylib"]`と指定することを求められますが、[wasm-packのチュートリアル](https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/template-deep-dive/cargo-toml.html#1-crate-type)によると、`wasm-pack test`のビルドをするためには`"rlib"`も指定する必要があります。

したがって、以下のように両方指定することで`wasm-pack test --headless --firefox`が動きます。
```toml
[lib]
# rlibもcrate-typeに加えないとtests/以下からインポートできない
crate-type = ["cdylib", "rlib"]
```

# まとめ・所感 #

フォームに入力された単語に対して辞書からサジェストさせる処理をRustで実装し、それをReactから呼び出すコードを書きました。たまたま発見したアルゴリズムを実装して、フロントエンドで重い処理がうまく高速に動いたのは嬉しかったです。Wasm上で大きなオブジェクトを作るような記事があまり見つからなかったので、今回取ったような`once_cell`を使う方法以外の方法をご存知だったらぜひ教えていただきたいです。

また、やり残しとしては以下を思いつきました。

  * 辞書のビルドを、パッケージのビルド時などに逃がすことで、`SubStringDictionary`の初期化にかかる６秒くらいを削減したい
  * 今回はやってみたかったのでRustを使ったが、ピュアにTypeScriptで実装しても性能が出たかどうか

# 参考 #

https://rustwasm.github.io/docs/wasm-pack/introduction.html

https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/template-deep-dive/cargo-toml.html#1-crate-type

[^1]: https://github.com/justinwilaby/spellchecker-wasm はこれを`wasm-bindgen`なしで行っている
