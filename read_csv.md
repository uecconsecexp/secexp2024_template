[README.md](README.md) > [q_and_a.md](q_and_a.md) > read_csv.md

文責: 2023年度TA 修士2年 奥山

# CSVファイルの内容を[][]float64型配列に格納するハンズオン

省略を交えつつハンズオン形式でソースコードを書いていきます。Go言語自体の一般的な話も交えていきます。

まずは `import` 文でパッケージを追加します。

```go:lib.go
package contentssecurity

import (
	"encoding/csv" // <- 追加
	"fmt"
	"os" // <- 追加

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