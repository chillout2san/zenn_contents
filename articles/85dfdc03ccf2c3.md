---
title: "Go言語で学ぶUTF-8"
emoji: "🐁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go"]
published: true
---

# 執筆のきっかけ

『Go言語でつくるインタプリタ』という本を読んでいました。(面白いのでオススメ)
字句解析のパートで文字列を操作する箇所があり、そういえばUTF-8って何だっけと思ったのでまとめます。

https://www.oreilly.co.jp/books/9784873118222/

:::message
以下Qiita記事がとても分かりやすいです。
車輪の再発明になりますが、自分の整理のために書いてみました。
:::

https://qiita.com/seihmd/items/4a878e7fa340d7963fee

# Unicodeについて

UTF-8について整理する前にUnicodeについて触れます。
普段何気なく漢字やひらがなをタイピングしていますが、パソコンはどうこれらの文字を解釈しているのでしょうか。

> Unicode とは、世界の様々な言語、書式、記号に、番号を割り当てて定義した標準の文字コード です。一つ一つの文字 に番号を割り当てることで、プログラマーは、どの言語が混ざっていても、コンピューターに保存、処理、伝送させるような文字エンコーディングを同じファイルやプログラムの中に作ることができます。

https://developer.mozilla.org/ja/docs/Glossary/Unicode

Unicodeは直訳すると「単一のコード」になります。
平たく言えば、世界中の言語や記号を文字列にしたものです。
例えば`a`はUnicodeでは`U+0061`になり、このコードをコードポイントと呼んだりします。
ですが`U+0061`という文字列をパソコンはまだ解釈できません。

# UTF-8って何？

> UTF-8 (UCS Transformation Format 8) は World Wide Web において最も一般的な文字エンコーディングです。 1 文字あたり 1 ～ 4 バイトで表します。UTF-8 は ASCII に対して後方互換性を持っており、すべての標準 Unicode 文字を表現することができます。

https://developer.mozilla.org/ja/docs/Glossary/UTF-8

> エンコーディングはバイト列と文字を対応付けるものです。バイトの並びは文字としてさまざまに解釈できます。特定のエンコーディング（UTF-8 など）を設定することで、バイトの並びがどのように解釈されるかを定めることができます。

https://developer.mozilla.org/ja/docs/Glossary/character_encoding

文字列とバイト列を紐付けるルールの一つがUTF-8です。
これでUnicodeがバイト列になり、パソコンが理解できるようになりました。
UTF-8は1文字(つまり一つのコードポイント)を1〜4バイトで表現します。
そしてGo言語では文字列をUTF-8のバイト列で扱います。

:::message
文字列についての詳しい話はGo公式ブログが参考になります。
:::

https://go.dev/blog/strings

# 実際にやってみた

ではGoで実際に見てみましょう。

- `a`という文字列をbyte配列でキャストして出力してみましょう。`[97]`と長さが1のバイト列が出力されました。すなわち`a`は1バイトで表現されます。

```go
func main() {
    str := "a"
    fmt.Println([]byte(str)) // [97]
}
```

- `あ`で同じことやってみると3バイトです。

```go
func main() {
    str := "あ"
    fmt.Println([]byte(str)) // [227 129 130]
}
```

- `👈`でやってみると4バイトです。

```go
func main() {
    str := "👈"
    fmt.Println([]byte(str)) // [240 159 145 136]
}
```

ここで注意が必要なのは今表示されたバイト列は10進数で表されているということです。
前述のとおり`a`は10進数だと`[97]`でしたので、16進数にしてみましょう。
`fmt.Sprintf("%x", 97)`で16進数の数を出せます。

```go
func main() {
    hoge := fmt.Sprintf("%x", 97)
    fmt.Println(hoge) // 61
}
```

Goでは2進数のリテラルはないですが、8進数・16進数リテラルはあるようです。

https://ashitani.jp/golangtips/tips_num.html

`a`を16進数のバイト列で表現してみます。

```go
// 0xつけて16進数のリテラルとして使ってみる
func main() {
    fmt.Println(string([]byte{0x67})) // a
}
```

# どういう時に役に立つのか

UnicodeやUTF-8を理解しておくと、文字列の長さを数える時の理解に役立ちます。
`あ`という文字の長さを取得しようと、`len`関数に渡してみましたが、`3`と出力されてしまいました。

```go
func main() {
    fmt.Println(len("あ")) // 3
}
```

`len`メソッドのコメントを見てみると、引数がstring型の場合はバイトの長さを返却するとあります。
先程見たように`あ`は長さ3のバイト列なので、`3`が出力されたわけですね。

```go:builtin/builtin.go
// The len built-in function returns the length of v, according to its type:
//
//	Array: the number of elements in v.
//	Pointer to array: the number of elements in *v (even if v is nil).
//	Slice, or map: the number of elements in v; if v is nil, len(v) is zero.
//	String: the number of bytes in v.
//	Channel: the number of elements queued (unread) in the channel buffer;
//	         if v is nil, len(v) is zero.
//
// For some arguments, such as a string literal or a simple array expression, the
// result can be a constant. See the Go language specification's "Length and
// capacity" section for details.
func len(v Type) int
```

ここでやりたいことはあくまで文字列の長さを知りたいわけで、バイトの長さを知りたいわけではありません。
ここである文字はコードポジションと一対一で対応することを思い出してください。
コードポジションの数を数えれば文字列の長さを数えられそうです。
Goではコードポジションを短く扱うことが出来るruneというものがあります。

:::message
前述のとおり、UTF-8は1〜4バイトで文字列を表現します。
一番長いバイト列は4バイト、1バイトは8ビットなので、8bitが4つで32bit必要です。
なのでGoではruneはint32のエイリアスになっています。
:::

なので1バイトを超える文字列はrune配列にキャストしてから長さを数えると、意図通りの結果になります。

```go
func main() {
    r := []rune("あ")
    fmt.Println(len(r)) // 1
}
```

ただいちいちキャストするのは手間なので標準パッケージでメソッドが用意されています。
ここまでの話が理解できるとメソッドの名前も腑に落ちますね。

```go
func main() {
    fmt.Println(utf8.RuneCountInString("あ")) // 1
}
```

ちなみに文字列はバイト配列なので、ループを回したり添字でアクセスすることが出来ます。

```go
func main() {
    s := "あいう" // 全て3バイト文字
    fmt.Println(len(s)) //9

    // rangeで回すとruneが取れる
    for _, r := range s {
        fmt.Println(r) // 12354 12356 12358
    }

    // インデックスでアクセスするとbyteが取れる。
    // 0-3は先頭から3つ取ってこいという意味。なので「あ」が取得できる。
    fmt.Println(s[0:3]) // あ

    // こうすると3バイト文字を表現するのに2バイトしかないので表示がバグる。
    fmt.Println(s[0:2]) 

    // バイト配列で出してね、とするとバイトになる。
    fmt.Println([]byte(s[0:2])) // [227 129]
}
```

# おまけ

麻雀牌の`東`をGoで出力してみました。Unicodeのカバー範囲すごいですね。

```go
func main() {
    fmt.Println(string([]byte{0xF0, 0x9F, 0x80, 0x80})) // 🀀
}
```

文字コード表見てるとなかなか楽しいですね。

https://orange-factory.com/sample/utf8/code4/f09f.html#MahjongTiles
