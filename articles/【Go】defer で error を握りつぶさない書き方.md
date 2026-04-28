---
title: 【Go】defer で error を握りつぶさない書き方
emoji: 🕳️
type: tech
topics:
  - go
  - error
  - defer
published: false
tags:
  - go
  - error
  - defer
---

## 3行まとめ

1. `defer someFunc()` は `someFunc()` のエラーを戻り値に自動では反映しないため、`Close()` や `Rollback()` のエラーが黙って捨てられがちです。
2. 名前付き戻り値 `(err error)` を使うと、`return` 後に走る `defer` から戻り値そのものを更新できます。
3. Go 1.20 以降の `errors.Join` を使えば、主処理と後処理のエラーを「片方を消さずに」簡潔に合成できます。

## 1. はじめに

Go では `defer` を使って、関数終了時の後処理を書くことが多いです。`file.Close()` や `tx.Rollback()`、`rows.Close()` などは、ほとんどの Go プログラマが日常的に `defer` で呼び出しているのではないでしょうか。

しかし、`defer` で呼び出す関数が `error` を返す場合、そのエラーは自動では戻り値に反映されません。つまり、後処理のエラーが黙って捨てられる可能性があります。

この記事では、なぜそうなるのかを Go の仕様に沿って順に説明し、どう対処すればよいかを整理します。

**前提バージョン**: 第3節以降で登場する `errors.Join` は Go 1.20 以降で使えます。それ以前のバージョン向けには、第3節で `fmt.Errorf` による代替パターンも示します。

**対象読者**:

- `defer` は使っているが、戻り値との関係に自信がない方
- `Close()` や `Rollback()` のエラーをどう扱うべきか迷っている方
- 名前付き戻り値の実用的な使いどころを知りたい方

## 2. defer した関数の error は自動では返らない

まず、よく見かけるコードから始めます。

```go
// file.Close() は error を返すが、この書き方では戻り値は捨てられる
func writeFile(file *os.File) error {
	defer file.Close()

	return writeSomething(file)
}
```

`file.Close()` は `error` を返す関数ですが、`defer file.Close()` と書いた場合、その戻り値は受け取られずに捨てられます。

`defer` は、関数呼び出しを関数終了直前まで遅延させる仕組みです。遅延させた関数の戻り値を、自動で呼び出し元の戻り値に反映する仕組みは持っていません。

エラーを検知したい場合は、クロージャで明示的に受け取る必要があります。

```go
func writeFile(file *os.File) error {
	defer func() {
		if err := file.Close(); err != nil {
			log.Printf("file.Close failed: %v", err)
		}
	}()

	return writeSomething(file)
}
```

ただし、これでできることはログに残すことだけです。`file.Close()` のエラーを呼び出し元に返せてはいません。

では、ローカル変数 `err` を `defer` 内で更新すれば、エラーを戻り値に反映できるのでしょうか。次節で検証します。
## 3. ローカル変数 err を defer で更新しても戻り値に反映されない

次のコードを見てください。一見、`cleanup()` のエラーを戻り値に反映できそうに見えます。

```go
// 一見うまくいきそうに見えるが、cleanup のエラーは戻り値に反映されない
func f() error {
	var err error

	defer func() {
		if cleanupErr := cleanup(); cleanupErr != nil {
			err = cleanupErr
		}
	}()

	err = doMainWork()
	return err
}
```

しかし、このコードでは `cleanup()` のエラーは戻り値に反映されません。

理由は、`return err` の時点で戻り値が確定するためです。`defer` はその後に実行されますが、すでに確定した戻り値は変わりません。

概念的には、次のように読み替えると理解しやすくなります。

```go
func f() error {
	var err error

	defer func() {
		if cleanupErr := cleanup(); cleanupErr != nil {
			err = cleanupErr // err は更新されるが...
		}
	}()

	err = doMainWork()

	ret := err       // ← return err はここで戻り値を確定させる
	// defer が実行される（err は更新されるが ret は変わらない）
	return ret       // ← 返るのは ret
}
```

ポイントをまとめます。

- `return err` は `err` への参照を返すのではなく、その時点の `err` の値を評価して戻り値に設定します
- `defer` は `return` 式の評価後に実行されます
- `defer` 内でローカル変数 `err` を更新しても、すでに評価済みの戻り値には影響しません
- `return nil` の場合も同様です

では、戻り値そのものを `defer` から更新するにはどうすればよいのでしょうか。ここで名前付き戻り値が登場します。
## 4. 名前付き戻り値なら defer から戻り値を更新できる

名前付き戻り値を使うと、`return` 式の評価後に実行される `defer` から、戻り値そのものを更新できます。

### return doMainWork() の場合

```go
func f() (err error) {
	defer func() {
		if cleanupErr := cleanup(); cleanupErr != nil {
			if err != nil {
				err = fmt.Errorf("main: %w; cleanup: %v", err, cleanupErr)
				return
			}
			err = cleanupErr
		}
	}()

	return doMainWork()
}
```

処理の流れは次のとおりです。

1. `defer` が登録される
2. `doMainWork()` が実行される
3. `doMainWork()` の戻り値が名前付き戻り値 `err` に代入される
4. 関数を抜ける直前に `defer` が実行される
5. `cleanup()` が失敗していれば `err` を更新する
6. 更新後の `err` が呼び出し元に返る

概念的には、次のように読み替えられます。

```go
func f() (err error) {
	defer func() {
		if cleanupErr := cleanup(); cleanupErr != nil {
			if err != nil {
				err = fmt.Errorf("main: %w; cleanup: %v", err, cleanupErr)
				return
			}
			err = cleanupErr
		}
	}()

	err = doMainWork()  // ← return doMainWork() は、まず名前付き戻り値 err に代入する
	// ここで defer が実行される
	// err は名前付き戻り値そのものなので、defer 内の更新がそのまま戻り値に反映される
	return               // ← 更新後の err が返る
}
```

第2節の疑似コードと比較すると、違いが見えてきます。第2節では `ret := err` というコピーが介在していたため、`defer` 内の更新が戻り値に届きませんでした。名前付き戻り値では `err` 自身が戻り値スロットであるため、コピーが発生しません。`defer` が `err` を更新すれば、それがそのまま呼び出し元に返ります。

### return err の場合

実務では、主処理の前後に複数の処理がある場合、`return err` と書くことも多いです。名前付き戻り値であれば、この形でも `defer` からの更新は反映されます。

```go
func writeFile(name string, data []byte) (err error) {
	file, err := os.Create(name)
	if err != nil {
		return err
	}

	defer func() {
		if cleanupErr := file.Close(); cleanupErr != nil {
			if err != nil {
				err = fmt.Errorf("write: %w; close: %v", err, cleanupErr)
				return
			}
			err = cleanupErr
		}
	}()

	_, err = file.Write(data)
	return err  // ← 名前付き戻り値なのでこの形でも defer の更新が反映される
}
```

ここで、第2節の「ローカル変数 `err` の場合」との違いを明確にしておきます。

第2節のコードでは `func f() error`（無名の戻り値）でした。この場合、`err` は関数内のローカル変数にすぎません。`return err` は `err` の値を戻り値スロットにコピーするため、その後の `defer` で `err` を更新しても、すでにコピー済みの戻り値には届きません。

一方、名前付き戻り値の `err` は戻り値スロットそのものです。`return err` は「`err` の値を `err`（自分自身）に代入する」という実質ノーオペレーションになり、`defer` 内の更新がそのまま反映されます。

2つを並べてみると、違いがはっきりします。

```go
// NG: err はローカル変数。return err で値がコピーされ、defer の更新は届かない
func f() error {
	var err error
	defer func() { err = errors.Join(err, cleanup()) }()
	err = doMainWork()
	return err
}

// OK: err は名前付き戻り値。err 自身が戻り値スロットなので、defer の更新がそのまま反映される
func f() (err error) {
	defer func() { err = errors.Join(err, cleanup()) }()
	err = doMainWork()
	return err
}
```

違いは `func f() error` か `func f() (err error)` かだけです。コードの見た目はほぼ同じですが、`err` が指すものがまったく異なります。

- `func f() error` の `err` → ローカル変数。`return err` で値がコピーされて終わり
- `func f() (err error)` の `err` → 戻り値スロットそのもの。`defer` 内の更新がそのまま呼び出し元に返る

### ケース別の動作

名前付き戻り値を使った場合、主処理と後処理の成否の組み合わせに応じて、最終的な戻り値は次のようになります。

|doMainWork()|cleanup()|最終的な戻り値|
|---|---|---|
|成功 (nil)|成功 (nil)|nil|
|失敗 (err)|成功 (nil)|doMainWork のエラー|
|成功 (nil)|失敗 (err)|cleanup のエラー|
|失敗 (err)|失敗 (err)|両方を含むエラー|

すべてのケースで、エラーが握りつぶされることなく呼び出し元に伝わります。

### 無条件更新は危険

ただし、注意すべき落とし穴があります。次のように `defer` 内で無条件に `err = cleanup()` と書くと、主処理のエラーを消してしまいます。

```go
// NG: cleanup が nil を返すと doMainWork のエラーが消える
func f() (err error) {
	defer func() {
		err = cleanup()
	}()

	return doMainWork()
}
```

`doMainWork()` が失敗していても、`cleanup()` が成功すると `err` が `nil` で上書きされます。後処理エラーを扱うつもりが、主処理エラーを消すバグになってしまうのです。

前述の `if` 分岐パターンはこの問題を回避していますが、分岐がやや冗長です。Go 1.20 以降では `errors.Join` を使うことで、この分岐を簡潔に書けます。

## 5. errors.Join で主処理と後処理のエラーを合成する（Go 1.20+）

前節の `if` 分岐を `errors.Join` で置き換えると、次のようにさらに簡潔に書けます。

まず、前節のコードを再掲します。

```go
// 前節のコード（比較用）
func f() (err error) {
	defer func() {
		if cleanupErr := cleanup(); cleanupErr != nil {
			if err != nil {
				err = fmt.Errorf("main: %w; cleanup: %v", err, cleanupErr)
				return
			}
			err = cleanupErr
		}
	}()

	return doMainWork()
}
```

`errors.Join` を使うと、同じことが一行で書けます。

```go
// errors.Join を使った書き方
func f() (err error) {
	defer func() {
		err = errors.Join(err, cleanup())
	}()

	return doMainWork()
}
```

なぜこれで前節の分岐と同じ動作になるのでしょうか。`errors.Join` は、すべての引数が `nil` のときだけ `nil` を返すという性質を持っています。

|errors.Join の引数|戻り値|
|---|---|
|`errors.Join(nil, nil)`|`nil`|
|`errors.Join(mainErr, nil)`|`mainErr`|
|`errors.Join(nil, cleanupErr)`|`cleanupErr`|
|`errors.Join(mainErr, cleanupErr)`|両方を含むエラー|

この性質により、前節で書いた4つの分岐がすべて自然に処理されます。主処理のエラーを消さず、後処理のエラーも捨てず、両方 `nil` なら `nil` を返してくれます。

`errors.Join` を使うメリットをまとめます。

- 主処理のエラーを消さない
- 後処理のエラーも捨てない
- `nil` を自然に扱える
- 分岐が減り、コードの意図が明確になる
- `errors.Is` / `errors.As` で、合成されたエラーに含まれる個々のエラーを判定できる

注意点もあります。`errors.Join` が返すエラーのメッセージは、デフォルトでは改行区切りになります。たとえば、次のようなコードを実行した場合の `Error()` の出力は以下のようになります。

```go
err := errors.Join(
    errors.New("main work failed"),
    errors.New("cleanup failed"),
)
fmt.Println(err.Error())
```

```
main work failed
cleanup failed
```

`%v` でフォーマットしたときも同じ文字列（`"main work failed\ncleanup failed"`）になるため、ログ出力のフォーマットによっては読みにくくなる場合があります。

エラーメッセージに文脈を加えたい場合は、`fmt.Errorf` と組み合わせることもできます。

```go
defer func() {
	if cleanupErr := cleanup(); cleanupErr != nil {
		err = errors.Join(err, fmt.Errorf("cleanup: %w", cleanupErr))
	}
}()
```

最後にひとつ補足しておきます。`errors.Join` は、名前付き戻り値の仕組みを置き換えるものではありません。`defer` から戻り値を更新するには、あくまで名前付き戻り値が必要です。`errors.Join` は、主処理と後処理のエラー合成を簡潔にするための道具です。

## 6. 実務での判断基準と避けたい書き方

ここまで、`defer` のエラーを戻り値に反映する方法を見てきました。しかし、すべての後処理エラーを戻り値に含めるべきかというと、そうではありません。処理の性質に応じて判断する必要があります。

### 判断軸

判断の主軸は「呼び出し元がそのエラーで何か対処できるか」です。

| No.  | 後処理エラーの性質                    | 呼び出し元の対処必要性 | 推奨                      |
| ---- | ---------------------------- | ----------- | ----------------------- |
| ケース1 | データ整合性に影響する（flush, rename 等） | 対処必要        | 名前付き戻り値 + `errors.Join` |
| ケース2 | 観測・調査に必要だが主処理結果に影響しない        | 対処不要        | `defer` 内でログ出力          |
| ケース3 | エラーを返さない、または失敗が実質的に無害        | 対処不要        | `defer someFunc()` のまま  |

### ケース1: 後処理エラーを返すべき例

典型的なのは、ファイル書き込み後の `Close()` です。`Write()` の時点ではカーネルのページキャッシュに書き込んだだけで、実際のディスク I/O は非同期に行われることが一般的です。そのため、書き込み自体のエラーが `close(2)` の時点で初めて報告されることがあります。これはローカルファイルシステムでも、NFS のようなネットワークファイルシステムでも起こり得ます。

詳しくは [Linux の `close(2)` man page](https://man7.org/linux/man-pages/man2/close.2.html) や [Go の `os.File.Close` ドキュメント](https://pkg.go.dev/os#File.Close) を参照してください。

```go
func writeFile(name string, data []byte) (err error) {
	file, err := os.Create(name)
	if err != nil {
		return err
	}

	defer func() {
		err = errors.Join(err, file.Close())
	}()

	_, err = file.Write(data)
	return err
}
```

この関数では名前付き戻り値を使っているため、`return err` であっても `defer` 内の `errors.Join` による更新が戻り値に反映されます（第3節で解説したとおりです）。

同様に、一時ファイルの rename や remove など、後処理の成否が外部状態に影響するケースでも、エラーを返すべきです。

### ケース2: ログだけでよい例

後処理の失敗が主処理の結果を変えない場合や、呼び出し元がそのエラーでリカバリできない場合は、ログに残すだけで十分です。

```go
func f() error {
	defer func() {
		if err := cleanup(); err != nil {
			log.Printf("cleanup failed: %v", err)
		}
	}()

	return doMainWork()
}
```

### 避けたい書き方

この記事で扱ってきた内容をもとに、避けたい書き方をまとめます。

**パターン A: ローカル変数を更新しても戻り値に反映されない**

第2節で仕組みを解説しました。`func f() error` のローカル変数 `err` を `defer` 内で更新しても、`return err` 後の戻り値には反映されません。

```go
// NG: cleanup のエラーは戻り値に反映されない
func f() error {
	var err error

	defer func() {
		err = errors.Join(err, cleanup())
	}()

	err = doMainWork()
	return err
}
```

**パターン B: 無条件更新で主処理エラーを消す**

第3節で仕組みを解説しました。`err = cleanup()` と無条件に書くと、`cleanup()` が成功した場合に `err` が `nil` で上書きされ、主処理のエラーが消えます。

```go
// NG: cleanup が成功すると doMainWork のエラーが消える
func f() (err error) {
	defer func() {
		err = cleanup()
	}()

	return doMainWork()
}
```

### 名前付き戻り値を使う際の注意

名前付き戻り値は、短く意図が明確な関数では有効な手法です。しかし、長い関数で多用すると、どこで戻り値が変更されるか追いにくくなります。`defer` で戻り値を更新する明確な理由がある場合に限って使うのがよいでしょう。

もうひとつ注意したいのが、`:=` によるシャドーイングです。

```go
func f() (err error) {
	defer func() {
		err = errors.Join(err, cleanup())
	}()

	// NG: if スコープ内で := を使うと、名前付き戻り値 err が隠れる
	// この err は名前付き戻り値とは別の変数。ここで握りつぶされて外に伝わらない
	if result, err := doSomething(); err != nil {
		_ = result
		log.Printf("doSomething failed: %v", err)
		// ここで外側の err に代入し忘れると、このエラーは消える
	}

	return // ← 名前付き戻り値 err は nil のまま。defer の cleanup エラーだけが返る
}
```

`if` ブロック内で `:=` を使うと、新しいスコープで別の `err` 変数が宣言されます。この `err` は名前付き戻り値の `err` とは別の変数なので、`if` ブロックを抜けるとその値はどこにも残りません。`defer` が参照する `err`（名前付き戻り値）は、`if` ブロック内の変更を知らないため、`doSomething()` のエラーは握りつぶされてしまいます。

なお、`return err` のように明示的に値を指定して return した場合は、`return` 式の評価時にシャドーイングされた `err` の値が名前付き戻り値スロットへコピーされるため、エラーは伝わります。問題になるのは、上記のような「シャドーイング後に裸の `return`」や「シャドーイング後に外の `err` への代入を忘れる」ケースです。

このようなシャドーイングは静的解析で検出できます。標準の `go vet` ではデフォルトで shadow 検出は有効になっておらず、別途 [`go.googlesource.com/tools/...`](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/shadow) の `shadow` analyzer を入れて `go vet -vettool=...` で走らせる必要があります。実用上は、[`golangci-lint`](https://golangci-lint.run/) の `govet` プラグインで `shadow` を有効化するのが手軽です。
## 7. まとめ

この記事で扱った内容を振り返ります。

- `defer someFunc()` は、`someFunc()` のエラーを自動では返しません
- ローカル変数 `err` を `defer` 内で更新しても、`return err` 後の戻り値には反映されません
- 名前付き戻り値を使うと、`defer` 内から戻り値そのものを更新できます
- Go 1.20 以降では、`errors.Join` で主処理と後処理のエラーを簡潔に合成できます
- 後処理エラーを戻り値に含めるかどうかは、呼び出し元が対処できるかで判断します

後処理エラーも返したい場合の基本形は次のとおりです。

```go
func f() (err error) {
	defer func() {
		err = errors.Join(err, cleanup())
	}()

	return doMainWork()
}
```
