---
title: "Visitor パターン"
---

# Visitor パターン

## 概要

Visitorパターンは、データ構造と操作を分離し、構造を変更せずに新しい操作を追加できるようにするパターンです。

## 問題

以下のような状況で使用を検討します：

- データ構造は安定しているが、操作が頻繁に追加される
- 異なる型のオブジェクトに対して統一的な操作を行いたい
- オブジェクトの型に応じた処理を行いたい

## 構造

```
┌───────────────┐       ┌───────────────┐
│    Element    │◀──────│    Visitor    │
│  (interface)  │       │  (interface)  │
├───────────────┤       ├───────────────┤
│ + Accept(v)   │       │ + VisitA(a)   │
└───────────────┘       │ + VisitB(b)   │
        △               └───────────────┘
   ┌────┴────┐                  △
┌──────┐  ┌──────┐         ┌────┴────┐
│Elem A│  │Elem B│    ┌────────┐ ┌────────┐
└──────┘  └──────┘    │Visitor1│ │Visitor2│
                      └────────┘ └────────┘
```

## 実装

### 図形の面積・周囲長計算

```go
package main

import (
    "fmt"
    "math"
)

// Shape は図形のインターフェース
type Shape interface {
    Accept(visitor ShapeVisitor)
}

// ShapeVisitor は図形のビジター
type ShapeVisitor interface {
    VisitCircle(c *Circle)
    VisitRectangle(r *Rectangle)
    VisitTriangle(t *Triangle)
}

// Circle は円
type Circle struct {
    Radius float64
}

func (c *Circle) Accept(v ShapeVisitor) {
    v.VisitCircle(c)
}

// Rectangle は長方形
type Rectangle struct {
    Width, Height float64
}

func (r *Rectangle) Accept(v ShapeVisitor) {
    v.VisitRectangle(r)
}

// Triangle は三角形
type Triangle struct {
    A, B, C float64 // 3辺の長さ
}

func (t *Triangle) Accept(v ShapeVisitor) {
    v.VisitTriangle(t)
}

// AreaVisitor は面積を計算するビジター
type AreaVisitor struct {
    TotalArea float64
}

func (v *AreaVisitor) VisitCircle(c *Circle) {
    area := math.Pi * c.Radius * c.Radius
    fmt.Printf("円の面積: %.2f\n", area)
    v.TotalArea += area
}

func (v *AreaVisitor) VisitRectangle(r *Rectangle) {
    area := r.Width * r.Height
    fmt.Printf("長方形の面積: %.2f\n", area)
    v.TotalArea += area
}

func (v *AreaVisitor) VisitTriangle(t *Triangle) {
    // ヘロンの公式
    s := (t.A + t.B + t.C) / 2
    area := math.Sqrt(s * (s - t.A) * (s - t.B) * (s - t.C))
    fmt.Printf("三角形の面積: %.2f\n", area)
    v.TotalArea += area
}

// PerimeterVisitor は周囲長を計算するビジター
type PerimeterVisitor struct {
    TotalPerimeter float64
}

func (v *PerimeterVisitor) VisitCircle(c *Circle) {
    perimeter := 2 * math.Pi * c.Radius
    fmt.Printf("円の周囲長: %.2f\n", perimeter)
    v.TotalPerimeter += perimeter
}

func (v *PerimeterVisitor) VisitRectangle(r *Rectangle) {
    perimeter := 2 * (r.Width + r.Height)
    fmt.Printf("長方形の周囲長: %.2f\n", perimeter)
    v.TotalPerimeter += perimeter
}

func (v *PerimeterVisitor) VisitTriangle(t *Triangle) {
    perimeter := t.A + t.B + t.C
    fmt.Printf("三角形の周囲長: %.2f\n", perimeter)
    v.TotalPerimeter += perimeter
}

func main() {
    shapes := []Shape{
        &Circle{Radius: 5},
        &Rectangle{Width: 4, Height: 6},
        &Triangle{A: 3, B: 4, C: 5},
    }

    fmt.Println("=== 面積計算 ===")
    areaVisitor := &AreaVisitor{}
    for _, shape := range shapes {
        shape.Accept(areaVisitor)
    }
    fmt.Printf("合計面積: %.2f\n", areaVisitor.TotalArea)

    fmt.Println("\n=== 周囲長計算 ===")
    perimeterVisitor := &PerimeterVisitor{}
    for _, shape := range shapes {
        shape.Accept(perimeterVisitor)
    }
    fmt.Printf("合計周囲長: %.2f\n", perimeterVisitor.TotalPerimeter)
}
```

### AST（抽象構文木）の例

```go
package main

import "fmt"

// Node はASTノード
type Node interface {
    Accept(visitor NodeVisitor)
}

type NodeVisitor interface {
    VisitNumber(n *NumberNode)
    VisitBinaryOp(n *BinaryOpNode)
}

// NumberNode は数値ノード
type NumberNode struct {
    Value int
}

func (n *NumberNode) Accept(v NodeVisitor) {
    v.VisitNumber(n)
}

// BinaryOpNode は二項演算ノード
type BinaryOpNode struct {
    Left, Right Node
    Operator    string
}

func (n *BinaryOpNode) Accept(v NodeVisitor) {
    v.VisitBinaryOp(n)
}

// PrintVisitor は式を文字列で出力
type PrintVisitor struct {
    Result string
}

func (v *PrintVisitor) VisitNumber(n *NumberNode) {
    v.Result = fmt.Sprintf("%d", n.Value)
}

func (v *PrintVisitor) VisitBinaryOp(n *BinaryOpNode) {
    leftV := &PrintVisitor{}
    n.Left.Accept(leftV)
    rightV := &PrintVisitor{}
    n.Right.Accept(rightV)
    v.Result = fmt.Sprintf("(%s %s %s)", leftV.Result, n.Operator, rightV.Result)
}

// EvalVisitor は式を評価
type EvalVisitor struct {
    Result int
}

func (v *EvalVisitor) VisitNumber(n *NumberNode) {
    v.Result = n.Value
}

func (v *EvalVisitor) VisitBinaryOp(n *BinaryOpNode) {
    leftV := &EvalVisitor{}
    n.Left.Accept(leftV)
    rightV := &EvalVisitor{}
    n.Right.Accept(rightV)

    switch n.Operator {
    case "+":
        v.Result = leftV.Result + rightV.Result
    case "-":
        v.Result = leftV.Result - rightV.Result
    case "*":
        v.Result = leftV.Result * rightV.Result
    }
}

func main() {
    // (3 + 5) * 2 を表現
    ast := &BinaryOpNode{
        Left: &BinaryOpNode{
            Left:     &NumberNode{Value: 3},
            Right:    &NumberNode{Value: 5},
            Operator: "+",
        },
        Right:    &NumberNode{Value: 2},
        Operator: "*",
    }

    printVisitor := &PrintVisitor{}
    ast.Accept(printVisitor)
    fmt.Println("式:", printVisitor.Result)

    evalVisitor := &EvalVisitor{}
    ast.Accept(evalVisitor)
    fmt.Println("結果:", evalVisitor.Result)
}
```

## Goでのポイント

### 型スイッチでの代替

Goでは型スイッチでシンプルに実装することも可能です：

```go
func Calculate(shape any) float64 {
    switch s := shape.(type) {
    case *Circle:
        return math.Pi * s.Radius * s.Radius
    case *Rectangle:
        return s.Width * s.Height
    default:
        return 0
    }
}
```

ただし、新しい型を追加した場合にコンパイルエラーにならないため、Visitorパターンの方が安全な場合があります。

## トレードオフ

| メリット | デメリット |
|---------|----------|
| 新しい操作の追加が容易 | 新しい要素型の追加が困難 |
| 関連操作を一箇所にまとめられる | Double Dispatch が必要 |
| 要素の内部状態にアクセスできる | Visitor と Element の結合 |

## まとめ

- Visitorはデータ構造と操作を分離する
- 新しい操作を追加しやすいが、新しい型の追加は困難
- Goでは型スイッチとVisitorパターンを状況に応じて使い分ける

これで振る舞いに関するパターンの解説は終わりです。次章で本書のまとめを行います。
