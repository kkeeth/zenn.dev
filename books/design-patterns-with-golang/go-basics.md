---
title: "Go言語の基礎知識"
---

# Go言語の基礎知識

本章では、デザインパターンを理解するために必要なGoの基礎知識を確認します。

## 環境構築

### Goのインストール

公式サイト（https://go.dev/dl/）からインストーラーをダウンロードするか、パッケージマネージャーを使用します。

```bash
# macOS (Homebrew)
brew install go

# Ubuntu/Debian
sudo apt install golang

# Windows (Chocolatey)
choco install golang
```

バージョンを確認します：

```bash
go version
# go version go1.22.0 darwin/arm64
```

### プロジェクトの初期化

```bash
mkdir design-patterns-go
cd design-patterns-go
go mod init design-patterns-go
```

## 構造体（Struct）

Goにはクラスがありませんが、構造体を使ってデータと振る舞いをまとめることができます。

```go
package main

import "fmt"

// 構造体の定義
type User struct {
    ID   int
    Name string
    Age  int
}

// メソッドの定義（レシーバを使用）
func (u User) Greet() string {
    return fmt.Sprintf("こんにちは、%sです", u.Name)
}

// ポインタレシーバ（構造体を変更する場合）
func (u *User) SetAge(age int) {
    u.Age = age
}

func main() {
    user := User{ID: 1, Name: "田中", Age: 30}
    fmt.Println(user.Greet()) // こんにちは、田中です

    user.SetAge(31)
    fmt.Println(user.Age) // 31
}
```

### 値レシーバとポインタレシーバ

- **値レシーバ `(u User)`**: 構造体のコピーを受け取る。読み取り専用の操作に使用
- **ポインタレシーバ `(u *User)`**: 構造体のポインタを受け取る。フィールドを変更する場合に使用

## インターフェース

Goのインターフェースは、メソッドの集合を定義します。**暗黙的実装**により、インターフェースを実装することを明示的に宣言する必要がありません。

```go
package main

import "fmt"

// インターフェースの定義
type Speaker interface {
    Speak() string
}

// Dog構造体
type Dog struct {
    Name string
}

func (d Dog) Speak() string {
    return "ワンワン"
}

// Cat構造体
type Cat struct {
    Name string
}

func (c Cat) Speak() string {
    return "ニャー"
}

// インターフェースを受け取る関数
func MakeSound(s Speaker) {
    fmt.Println(s.Speak())
}

func main() {
    dog := Dog{Name: "ポチ"}
    cat := Cat{Name: "タマ"}

    MakeSound(dog) // ワンワン
    MakeSound(cat) // ニャー
}
```

### 空インターフェース

`interface{}` または `any`（Go 1.18以降）は、任意の型を受け取ることができます。

```go
func PrintAnything(v any) {
    fmt.Printf("値: %v, 型: %T\n", v, v)
}
```

## 埋め込み（Embedding）

Goには継承がありませんが、構造体の埋め込みで類似の機能を実現できます。

```go
package main

import "fmt"

// 基底となる構造体
type Animal struct {
    Name string
}

func (a Animal) Move() {
    fmt.Printf("%sが移動した\n", a.Name)
}

// Animalを埋め込んだ構造体
type Dog struct {
    Animal // 埋め込み
    Breed  string
}

func (d Dog) Bark() {
    fmt.Printf("%sが吠えた\n", d.Name)
}

func main() {
    dog := Dog{
        Animal: Animal{Name: "ポチ"},
        Breed:  "柴犬",
    }

    dog.Move() // Animalのメソッドを呼び出せる
    dog.Bark()
}
```

## コンストラクタパターン

Goには組み込みのコンストラクタがないため、`New`で始まる関数を作成する慣習があります。

```go
type Database struct {
    host     string
    port     int
    user     string
    password string
}

// コンストラクタ関数
func NewDatabase(host string, port int, user, password string) *Database {
    return &Database{
        host:     host,
        port:     port,
        user:     user,
        password: password,
    }
}

// デフォルト値を持つコンストラクタ
func NewDefaultDatabase() *Database {
    return &Database{
        host: "localhost",
        port: 5432,
    }
}
```

## 関数型

Goでは関数も第一級市民です。関数を変数に代入したり、引数として渡したりできます。

```go
package main

import "fmt"

// 関数型の定義
type Operation func(a, b int) int

func Add(a, b int) int {
    return a + b
}

func Multiply(a, b int) int {
    return a * b
}

// 関数を引数に取る高階関数
func Calculate(a, b int, op Operation) int {
    return op(a, b)
}

func main() {
    fmt.Println(Calculate(3, 4, Add))      // 7
    fmt.Println(Calculate(3, 4, Multiply)) // 12

    // 無名関数（クロージャ）
    subtract := func(a, b int) int {
        return a - b
    }
    fmt.Println(Calculate(10, 3, subtract)) // 7
}
```

## エラーハンドリング

Goでは例外ではなく、戻り値でエラーを返すのが一般的です。

```go
package main

import (
    "errors"
    "fmt"
)

func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("ゼロで除算することはできません")
    }
    return a / b, nil
}

func main() {
    result, err := Divide(10, 2)
    if err != nil {
        fmt.Println("エラー:", err)
        return
    }
    fmt.Println("結果:", result)
}
```

## ゴルーチンとチャネル

Goの並行処理機能です。一部のパターンで使用します。

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        fmt.Printf("Worker %d started job %d\n", id, j)
        time.Sleep(time.Second)
        fmt.Printf("Worker %d finished job %d\n", id, j)
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // 3つのワーカーを起動
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    // 5つのジョブを送信
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)

    // 結果を収集
    for a := 1; a <= 5; a++ {
        <-results
    }
}
```

## パッケージの可視性

Goでは、名前の先頭文字の大文字・小文字で可視性を制御します。

```go
package mypackage

// PublicStruct は外部パッケージからアクセス可能
type PublicStruct struct {
    PublicField  string // 外部からアクセス可能
    privateField string // 同一パッケージ内のみ
}

// PublicFunc は外部パッケージから呼び出し可能
func PublicFunc() {}

// privateFunc は同一パッケージ内のみ
func privateFunc() {}
```

## まとめ

本章で確認した要素は、デザインパターンの実装で頻繁に使用します：

- **構造体**: オブジェクトの表現
- **インターフェース**: 抽象化と多態性
- **埋め込み**: コンポジション
- **コンストラクタ関数**: オブジェクト生成
- **関数型**: 戦略の受け渡し

次章からは、いよいよデザインパターンの解説に入ります。まずは生成に関するパターンから始めましょう。
