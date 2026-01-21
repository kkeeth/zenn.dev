---
title: "Strategy パターン"
---

# Strategy パターン

## 概要

Strategyパターンは、アルゴリズムを定義し、それぞれをカプセル化して交換可能にするパターンです。アルゴリズムを利用するクライアントから独立して変更できます。

## 問題

以下のような状況で使用を検討します：

- 複数のアルゴリズムを使い分けたい
- アルゴリズムを実行時に切り替えたい
- 条件分岐を減らしたい

## 構造

```
┌───────────────┐       ┌───────────────┐
│    Context    │──────▶│   Strategy    │
├───────────────┤       │  (interface)  │
│ + Execute()   │       ├───────────────┤
│ - strategy    │       │ + Algorithm() │
└───────────────┘       └───────────────┘
                              △
                         ┌────┴────┐
                    ┌────────┐ ┌────────┐
                    │StratA │ │ StratB │
                    └────────┘ └────────┘
```

## 実装

### 基本的な実装（インターフェース）

```go
package main

import (
    "fmt"
    "sort"
)

// SortStrategy はソート戦略のインターフェース
type SortStrategy interface {
    Sort(data []int) []int
}

// BubbleSort はバブルソート
type BubbleSort struct{}

func (s *BubbleSort) Sort(data []int) []int {
    result := make([]int, len(data))
    copy(result, data)
    n := len(result)
    for i := 0; i < n-1; i++ {
        for j := 0; j < n-i-1; j++ {
            if result[j] > result[j+1] {
                result[j], result[j+1] = result[j+1], result[j]
            }
        }
    }
    return result
}

// QuickSort はクイックソート
type QuickSort struct{}

func (s *QuickSort) Sort(data []int) []int {
    result := make([]int, len(data))
    copy(result, data)
    sort.Ints(result)
    return result
}

// Sorter はソーターコンテキスト
type Sorter struct {
    strategy SortStrategy
}

func (s *Sorter) SetStrategy(strategy SortStrategy) {
    s.strategy = strategy
}

func (s *Sorter) Sort(data []int) []int {
    return s.strategy.Sort(data)
}

func main() {
    data := []int{64, 34, 25, 12, 22, 11, 90}

    sorter := &Sorter{}

    sorter.SetStrategy(&BubbleSort{})
    fmt.Println("BubbleSort:", sorter.Sort(data))

    sorter.SetStrategy(&QuickSort{})
    fmt.Println("QuickSort:", sorter.Sort(data))
}
```

### 関数型での実装（Goで最も一般的）

```go
package main

import "fmt"

// Strategy は戦略関数の型
type Strategy func(a, b int) int

// Calculator は計算機
type Calculator struct {
    strategy Strategy
}

func (c *Calculator) SetStrategy(s Strategy) {
    c.strategy = s
}

func (c *Calculator) Calculate(a, b int) int {
    return c.strategy(a, b)
}

// 戦略関数
func Add(a, b int) int      { return a + b }
func Subtract(a, b int) int { return a - b }
func Multiply(a, b int) int { return a * b }

func main() {
    calc := &Calculator{}

    calc.SetStrategy(Add)
    fmt.Println("Add:", calc.Calculate(10, 5))

    calc.SetStrategy(Subtract)
    fmt.Println("Subtract:", calc.Calculate(10, 5))

    calc.SetStrategy(Multiply)
    fmt.Println("Multiply:", calc.Calculate(10, 5))

    // 無名関数も使える
    calc.SetStrategy(func(a, b int) int {
        return a / b
    })
    fmt.Println("Divide:", calc.Calculate(10, 5))
}
```

### 支払い処理の例

```go
package main

import "fmt"

// PaymentStrategy は支払い戦略
type PaymentStrategy interface {
    Pay(amount float64) error
}

type CreditCard struct {
    number string
    cvv    string
}

func (c *CreditCard) Pay(amount float64) error {
    fmt.Printf("クレジットカード(%s)で ¥%.0f を支払いました\n",
        c.number[len(c.number)-4:], amount)
    return nil
}

type PayPal struct {
    email string
}

func (p *PayPal) Pay(amount float64) error {
    fmt.Printf("PayPal(%s)で ¥%.0f を支払いました\n", p.email, amount)
    return nil
}

type BankTransfer struct {
    accountNumber string
}

func (b *BankTransfer) Pay(amount float64) error {
    fmt.Printf("銀行振込(%s)で ¥%.0f を支払いました\n",
        b.accountNumber, amount)
    return nil
}

type ShoppingCart struct {
    items   []string
    payment PaymentStrategy
}

func (c *ShoppingCart) SetPaymentMethod(p PaymentStrategy) {
    c.payment = p
}

func (c *ShoppingCart) Checkout(amount float64) error {
    return c.payment.Pay(amount)
}

func main() {
    cart := &ShoppingCart{items: []string{"商品A", "商品B"}}

    // クレジットカード払い
    cart.SetPaymentMethod(&CreditCard{number: "1234-5678-9012-3456", cvv: "123"})
    cart.Checkout(5000)

    // PayPal払い
    cart.SetPaymentMethod(&PayPal{email: "user@example.com"})
    cart.Checkout(3000)

    // 銀行振込
    cart.SetPaymentMethod(&BankTransfer{accountNumber: "1234567"})
    cart.Checkout(10000)
}
```

### 圧縮アルゴリズムの例

```go
package main

import "fmt"

type CompressionStrategy func(data []byte) []byte

func GzipCompress(data []byte) []byte {
    fmt.Println("Gzip圧縮を適用")
    return data // 実際には圧縮処理
}

func ZipCompress(data []byte) []byte {
    fmt.Println("Zip圧縮を適用")
    return data
}

func NoCompress(data []byte) []byte {
    fmt.Println("圧縮なし")
    return data
}

type FileCompressor struct {
    compress CompressionStrategy
}

func (f *FileCompressor) Compress(data []byte) []byte {
    return f.compress(data)
}

func main() {
    compressor := &FileCompressor{}
    data := []byte("sample data")

    compressor.compress = GzipCompress
    compressor.Compress(data)

    compressor.compress = ZipCompress
    compressor.Compress(data)
}
```

## Goでのポイント

### 標準ライブラリでの活用

`sort.Interface`はStrategyパターンの一例です：

```go
type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }

sort.Sort(ByAge(people))
```

### 関数オプションパターンとの組み合わせ

```go
type Options struct {
    Encoder func([]byte) []byte
    Logger  func(string)
}

func NewService(opts ...func(*Options)) *Service {
    options := &Options{
        Encoder: defaultEncoder,
        Logger:  defaultLogger,
    }
    for _, opt := range opts {
        opt(options)
    }
    return &Service{options: options}
}
```

## まとめ

- Strategyはアルゴリズムをカプセル化して交換可能にする
- Goでは関数型で非常にシンプルに実装できる
- 条件分岐を減らし、拡張性を高める

次章では、アルゴリズムの骨格を定義するTemplate Methodパターンを学びます。
