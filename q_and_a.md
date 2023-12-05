文責: 2023年度TA 修士2年 奥山

# よくある質問等

少し思想強めなTAが個人的に勝手にまとめたQ&Aページなので内容に偏りがあるかもしれません。

本ページには、実験自体を進めるにあたり必須な情報は載せていません。実験を進める上で各学生が自力でできていて欲しい内容が主となるページです。

随時追加していきます。

## 目次

- [どこにプログラムを書けばよいですか？](#どこにプログラムを書けばよいですか？)
- [CSVファイルの読み込みはどうすればよいですか？](#CSVファイルの読み込みはどうすればよいですか？)
- [通信がうまくいきません](#通信がうまくいきません)
- [どのようなコードが望ましいですか？](#どのようなコードが望ましいですか？)
- [ソースコード中にコメントを書いたほうがいいですか？](#ソースコード中にコメントを書いたほうがいいですか？)
- [その他](#その他)

## どこにプログラムを書けばよいですか？

特にこだわりがなければ `lib.go` に記述し、 `kadai〇/main.go` から読み込むように書いてください。

## CSVファイルの読み込みはどうすればよいですか？

後続で内容を使うことができれば基本的にはどのように読み込んでも構いませんが、特にこだわりがなく何か方針を求めるならば以下のようにしていただければと思います。

- `go run` コマンドを打つディレクトリから見た相対パスにしましょう。
    - `go run kadai1/main.go` とするならばプログラム中で必要なパスの文字列は `omomi.txt` や `./omomi.txt` 等になります。
    - `cd kadai1` としてから `go run main.go` として実行する場合は、 `../omomi.txt` になるでしょう。
    - `go build` で実行ファイルを作成した場合は、実行ファイルをパスの起点にします。
    - 絶対パスは避けた方が無難です。
- `os.Open` 関数を使うことでファイルオブジェクトを取得し、 `encoding/csv` ライブラリを使用して読み込みます。
    - 参考: https://pkg.go.dev/encoding/csv
    - 参考: https://zenn.dev/syo_yamamoto/articles/1fb502ef862490
- とりあえず、読み込んだ値は `[][]float64` 型の配列に入れることを目指しましょう。
    - もちろん何かしらの行列計算ライブラリの行列型として読み込める方法があるならばその方法を用いていただいてかまいません。
- 文字列は `strconv.ParseFloat` という関数を用いることで `float64` 型の数値にパースできます。

以下、省略を交えつつハンズオン形式でソースコードを書いていきます。一番最初のハンズオンになると思いますので、Go言語自体の一般的な話も交えていきます。

まずは `import` 文でパッケージを追加します。

```go:lib.go
package contentssecurity

import (
    "encoding/csv" // <- 追加
    "fmt"
    "os"      // <- 追加

    conn "github.com/uecconsecexp/secexp2022/se_go/connector"
)
```

ファイルを読み込む関数を定義していきます。 `main.go` ファイルから参照する場合はPublicにするため関数名の最初の1文字を大文字にします。`package contetntssecurity` としているファイル内からのみ参照する場合は小文字で始めても構いません。
(できるだけ小文字にしましょう。公開する関数は最低限にとどめた方が良いです。わからなければ大文字で良いです。ハンズオンでは大文字にします。)

- [Exported names](https://go-tour-jp.appspot.com/basics/3)

```go:lib.go
func CsvFile2Table(filename string) ([][]float64, error) {
	// ...
    // [][]float64型変数tableに読み込んだ結果を入れる
    // 最後までたどり着ければエラーはなかったものとしてnil (C言語で言うNULL) を返す

	return table, nil
}
```

Go言語では多値返却という方法でエラーを返すのがマナーです。

- [Multiple results](https://go-tour-jp.appspot.com/basics/6)
- [Errors](https://go-tour-jp.appspot.com/methods/19)

もし何事も無ければ　`return table, nil` という形で有効値を返します。

一方エラーが生じた時はそのエラーを `err` 変数に入れ、 `return nil, err` としてエラーを返します。

こうすると、`csvFile2Table` 関数を次のように呼び出したときにエラーかどうかわかるようになります。

```go:kadai1/main.go
func main() {
    // ...
	table, err := consec.CsvFile2Table("omomi.txt")
	if err != nil {
		panic(err)
	}
}
```

`main` 関数内で多値返却を受け取り、 `err` が `nil` かどうかでエラーかどうか判断します。 `nil` ではない時はエラー内容が入っていますので、適切にエラーハンドリングします。
今回は、一度切りの計算を行うだけですので、`main`関数では単に `panic` させてしまいましょう。ライブラリ内のどこで `panic` させるかは任せますが、基本的にはメインロジックを記述している場所だとデバッグがしやすいです。

importした `os` ライブラリを用いて、ファイルオブジェクトを取得していきます。

```go:lib.go
func CsvFile2Table(filename string) ([][]float64, error) {
	file, err := os.Open(filename)
	if err != nil {
		return nil, err
	}
	defer file.Close()

    // ...
}
```

C言語の場合と同様に、開いたファイルオブジェクトは読み終わったら基本的に閉じる必要があります。

本来は、関数の最後で `file.Close()` を呼び出して閉じますが、このような時に便利なのが `defer` 文です。

- [Defer](https://go-tour-jp.appspot.com/flowcontrol/12)

`defer` 文の後ろに書かれた処理はスタックされていき、関数の **終了時** に実行されます。
スタック(FILO)なので、最初に `defer` された `file.Close()` は最後にされます。この先で別な `defer` 文を記述した場合、先にそちらの処理が実行されることになります。

`file` に無事ファイルの内容が読み込めたら、 `csv.NewReader` 関数と `reader.ReadAll` メソッドを用いて、CSVとして読み込みます。

```go:lib.go
func CsvFile2Table(filename string) ([][]float64, error) {
	// ここまでで、ファイルの内容をfile変数に読み込んだ

	reader := csv.NewReader(file)
	str_table, err := reader.ReadAll()
	if err != nil {
		return nil, err
	}

    // ...
}
```

`err` が `nil` ではない場合(例えば`no such file or directory`等)、エラーを返しています。
[`reader.ReadAll` メソッド](https://pkg.go.dev/encoding/csv#Reader.ReadAll)のページ曰くこの関数は `[][]string` を返すので、`str_table` には文字列型の2次元配列が格納されています。ヘッダー(行頭 `[成績行列 国 数 英 理 社 内]` や行名 `生徒1` )が含まれてしまっていますので以降はそれを考慮しながら処理を書きます。

`str_table` を `[][]float64` の配列にしていきます。まずは、必要な大きさの動的配列を確保します。動的配列は `make` 関数を用いることで作成できます。

- [Creating a slice with make](https://go-tour-jp.appspot.com/moretypes/13)

長さはヘッダーを含めないため `len(str_table)-1` としている点に注意してください。

```go:lib.go
func CsvFile2Table(filename string) ([][]float64, error) {
	// ここまでで、ファイルの内容をstr_table変数に読み込んだ

    table := make([][]float64, len(str_table)-1)

    // ...
}
```

forループを回し、 `strconv.ParseFloat` 関数でパースしていきましょう。

Go言語のfor文では、`range`を使うことでインデックスと値を用いてループできます。

- [Range](https://go-tour-jp.appspot.com/moretypes/16)

また、ヘッダーを落とすためにスライス構文を利用しています。

- [Slices](https://go-tour-jp.appspot.com/moretypes/7)

```go:lib.go
func CsvFile2Table(filename string) ([][]float64, error) {
	// ここまでで、ファイルの内容をstr_table変数に読み込んだ

    table := make([][]float64, len(str_table)-1)
	for i, row := range str_table[1:] { // <- 1行目はヘッダーなので飛ばすため、[1:]というスライス
        // rowには一行分の[]stringが入っている
		table[i] = make([]float64, len(row)-1) // 各行に、動的配列を入れる
		for j, cell := range row[1:] { // <- 1列目は行名なので飛ばすため、[1:]というスライス
			// ここでtable[i][j]にfloat64の値を入れたい
		}
	}

    // ...
}
```

今、 `cell` という変数に文字列が格納されているので、後はこれを `strconv.ParseFloat` 関数でパースしましょう。

- 参考: [ParseFloat](https://pkg.go.dev/strconv#ParseFloat)

```go:lib.go
func CsvFile2Table(filename string) ([][]float64, error) {
	// 省略

	table := make([][]float64, len(str_table)-1)
	for i, row := range str_table[1:] {
		table[i] = make([]float64, len(row)-1)
		for j, cell := range row[1:] {
            // strconv.ParseFloatでパース
			table[i][j], err = strconv.ParseFloat(cell, 64)
            // エラーハンドリング
			if err != nil {
				return nil, err
			}
		}
	}

    // ...
}
```

これで目的の `table` が得られたので、後は多値返却するのみです。

```go:lib.go
func CsvFile2Table(filename string) ([][]float64, error) {
	// 省略

    return table, nil
}
```

## 通信がうまくいきません

- 相方のコードと `SendTable` や `ReceiveTable` のタイミングが合っているか確認しましょう。
- 通信自体をテストするための古いコードが残っている可能性があります。`yobikou.Receive()` や `chugaku.Send([]byte("ping"))` が残ってしまっていないかを良く確認しましょう。
- ソースコードを書き直したもののビルドしなおすのを忘れた、等の細かいミスにより上手くいっていない可能性もあります。今一度最初から丁寧に確認してみてください。

## どのようなコードが望ましいですか？

あまり色々指定したくはありませんが、強いて言えば次の2点を守っているコードは美しいなぁと感じます。

- 適切に関数が分離されている
    - 同じ処理を関数にまとめず、全部展開しているようなコードは読みにくいです。実行速度ではなく読みやすさを優先してください。
    - 一度しか呼び出されない処理でも、適切に関数に分けられていたほうが読みやすいです。
        - なんでもかんでも関数化しろということではありません。
        - 例えば逆行列計算をそのまま書かれてもどこまでが該当する処理かわかりにくい、という意味です。
- インデントが深くない
    - 無駄な条件分岐が多いと読みにくく感じます。
        - [早期リターン](https://zenn.dev/media_engine/articles/early_return)等を駆使してネストを減らしてみてください。
        - どうしても深くなってしまう場合、関数が適切に分けられていない場合が多いです。

## ソースコード中にコメントを書いたほうがいいですか？

ソースコードの読みやすさによって実装に関する点数が上下する可能性はありますが、コメントの有無で加点減点することはありません。

コメントをあれこれ考えている時間があるならば、関数名や変数名を工夫しプログラム自体を読みやすくしていただきたいです。

- 参考: [「コメントは書くな」 - Qiita](https://qiita.com/hiro-hori/items/1cf28a6ac75db077c8f3)

「書くな」とまでは言いませんが、余計なコメントが多いとむしろわかりにくくなってしまうのは確かです。

以下に、望ましいコメント例を載せます。

(編集中)

## その他

このページに載っていない内容で不明点があれば、なるべく直接TAに聞いてください。

直接聞くのが難しい時は

csec_exp(西暦)@oslab.inf.uec.ac.jp

にメールで聞いてください。開講期間中時間があれば回答いたします。

(2023年度なら csec_exp2023@oslab.inf.uec.ac.jp )

もしメールで聞く場合はなるべくソースコードを記載または添付してください。文章だけで尋ねられても答えかねる場合が多いです。