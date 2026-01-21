---
title: "Iterator パターン"
---

# Iterator パターン

## 概要

Iteratorパターンは、コレクションの内部構造を公開せずに、その要素に順次アクセスする方法を提供するパターンです。

## 問題

以下のような状況で使用を検討します：

- コレクションの内部構造を隠蔽したい
- 同じコレクションに対して複数の走査方法を提供したい
- コレクションの種類に依存しない統一的な走査を実現したい

## Goでのイテレーション

Go言語では、以下の方法でイテレーションを実現します：

1. **for-range**: スライス、マップ、チャネルに使用
2. **チャネル**: カスタムイテレータとして使用
3. **コールバック関数**: 関数を渡して各要素を処理
4. **Go 1.23のイテレータ**: `iter`パッケージ

## 実装

### チャネルを使った実装

```go
package main

import "fmt"

// Tree は二分木
type Tree struct {
    Value int
    Left  *Tree
    Right *Tree
}

// InOrder は中順走査（チャネルベースのイテレータ）
func (t *Tree) InOrder() <-chan int {
    ch := make(chan int)
    go func() {
        t.inOrderTraverse(ch)
        close(ch)
    }()
    return ch
}

func (t *Tree) inOrderTraverse(ch chan<- int) {
    if t == nil {
        return
    }
    t.Left.inOrderTraverse(ch)
    ch <- t.Value
    t.Right.inOrderTraverse(ch)
}

// PreOrder は前順走査
func (t *Tree) PreOrder() <-chan int {
    ch := make(chan int)
    go func() {
        t.preOrderTraverse(ch)
        close(ch)
    }()
    return ch
}

func (t *Tree) preOrderTraverse(ch chan<- int) {
    if t == nil {
        return
    }
    ch <- t.Value
    t.Left.preOrderTraverse(ch)
    t.Right.preOrderTraverse(ch)
}

func main() {
    //       4
    //      / \
    //     2   6
    //    / \ / \
    //   1  3 5  7
    tree := &Tree{
        Value: 4,
        Left: &Tree{
            Value: 2,
            Left:  &Tree{Value: 1},
            Right: &Tree{Value: 3},
        },
        Right: &Tree{
            Value: 6,
            Left:  &Tree{Value: 5},
            Right: &Tree{Value: 7},
        },
    }

    fmt.Print("中順走査: ")
    for v := range tree.InOrder() {
        fmt.Printf("%d ", v)
    }

    fmt.Print("\n前順走査: ")
    for v := range tree.PreOrder() {
        fmt.Printf("%d ", v)
    }
    fmt.Println()
}
```

### コールバック関数を使った実装

```go
package main

import "fmt"

type User struct {
    ID   int
    Name string
    Age  int
}

type UserCollection struct {
    users []User
}

func (c *UserCollection) Add(user User) {
    c.users = append(c.users, user)
}

// ForEach は各要素に対して関数を実行
func (c *UserCollection) ForEach(fn func(User)) {
    for _, user := range c.users {
        fn(user)
    }
}

// Filter は条件に合う要素を返す
func (c *UserCollection) Filter(predicate func(User) bool) []User {
    var result []User
    for _, user := range c.users {
        if predicate(user) {
            result = append(result, user)
        }
    }
    return result
}

// Map は各要素を変換
func (c *UserCollection) Map(fn func(User) string) []string {
    result := make([]string, len(c.users))
    for i, user := range c.users {
        result[i] = fn(user)
    }
    return result
}

func main() {
    collection := &UserCollection{}
    collection.Add(User{1, "田中", 25})
    collection.Add(User{2, "山田", 30})
    collection.Add(User{3, "佐藤", 22})
    collection.Add(User{4, "鈴木", 35})

    fmt.Println("=== ForEach ===")
    collection.ForEach(func(u User) {
        fmt.Printf("%s (%d歳)\n", u.Name, u.Age)
    })

    fmt.Println("\n=== Filter (25歳以上) ===")
    adults := collection.Filter(func(u User) bool {
        return u.Age >= 25
    })
    for _, u := range adults {
        fmt.Printf("%s (%d歳)\n", u.Name, u.Age)
    }

    fmt.Println("\n=== Map (名前のみ) ===")
    names := collection.Map(func(u User) string {
        return u.Name
    })
    fmt.Println(names)
}
```

### インターフェースベースの実装

```go
package main

import "fmt"

// Iterator はイテレータのインターフェース
type Iterator[T any] interface {
    HasNext() bool
    Next() T
}

// IntSliceIterator は整数スライスのイテレータ
type IntSliceIterator struct {
    data  []int
    index int
}

func NewIntSliceIterator(data []int) *IntSliceIterator {
    return &IntSliceIterator{data: data, index: 0}
}

func (it *IntSliceIterator) HasNext() bool {
    return it.index < len(it.data)
}

func (it *IntSliceIterator) Next() int {
    value := it.data[it.index]
    it.index++
    return value
}

// ReverseIterator は逆順イテレータ
type ReverseIterator struct {
    data  []int
    index int
}

func NewReverseIterator(data []int) *ReverseIterator {
    return &ReverseIterator{data: data, index: len(data) - 1}
}

func (it *ReverseIterator) HasNext() bool {
    return it.index >= 0
}

func (it *ReverseIterator) Next() int {
    value := it.data[it.index]
    it.index--
    return value
}

func main() {
    data := []int{1, 2, 3, 4, 5}

    fmt.Print("順方向: ")
    it := NewIntSliceIterator(data)
    for it.HasNext() {
        fmt.Printf("%d ", it.Next())
    }

    fmt.Print("\n逆方向: ")
    rit := NewReverseIterator(data)
    for rit.HasNext() {
        fmt.Printf("%d ", rit.Next())
    }
    fmt.Println()
}
```

### ページネーション

```go
package main

import "fmt"

type PaginatedIterator struct {
    items    []string
    pageSize int
    page     int
}

func NewPaginatedIterator(items []string, pageSize int) *PaginatedIterator {
    return &PaginatedIterator{items: items, pageSize: pageSize, page: 0}
}

func (p *PaginatedIterator) HasNextPage() bool {
    return p.page*p.pageSize < len(p.items)
}

func (p *PaginatedIterator) NextPage() []string {
    start := p.page * p.pageSize
    end := start + p.pageSize
    if end > len(p.items) {
        end = len(p.items)
    }
    p.page++
    return p.items[start:end]
}

func main() {
    items := []string{"A", "B", "C", "D", "E", "F", "G", "H"}
    paginator := NewPaginatedIterator(items, 3)

    pageNum := 1
    for paginator.HasNextPage() {
        fmt.Printf("Page %d: %v\n", pageNum, paginator.NextPage())
        pageNum++
    }
}
```

## Goでのポイント

### for-rangeの活用

```go
// スライス
for i, v := range slice { }

// マップ
for k, v := range m { }

// チャネル
for v := range ch { }
```

### iterパッケージ（Go 1.23+）

```go
func (c *Collection) All() iter.Seq[Item] {
    return func(yield func(Item) bool) {
        for _, item := range c.items {
            if !yield(item) {
                return
            }
        }
    }
}
```

## まとめ

- Iteratorはコレクションの走査方法を抽象化する
- Goではチャネル、コールバック、for-rangeで自然に実現
- Go 1.23以降は`iter`パッケージも利用可能

次章では、オブジェクト間の通信を仲介するMediatorパターンを学びます。
