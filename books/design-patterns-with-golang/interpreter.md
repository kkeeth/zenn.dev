---
title: "Interpreter パターン"
---

# Interpreter パターン

## 概要

Interpreterパターンは、言語の文法を定義し、その文法に従って文を解釈するパターンです。ドメイン固有言語（DSL）や式の評価に使用されます。

## 問題

以下のような状況で使用を検討します：

- シンプルな言語や文法を解釈したい
- 設定ファイルやルールエンジンを実装したい
- 数式や検索クエリを評価したい

## 構造

```
┌───────────────────┐
│    Expression     │
│   (interface)     │
├───────────────────┤
│ + Interpret(ctx)  │
└───────────────────┘
         △
    ┌────┴────┐
┌───────┐  ┌───────────┐
│Terminal│  │Non-Terminal│
│Expr    │  │Expr       │
└───────┘  └───────────┘
```

## 実装

### 数式の評価

```go
package main

import (
    "fmt"
    "strconv"
)

// Expression は式のインターフェース
type Expression interface {
    Interpret() int
}

// Number は数値（終端式）
type Number struct {
    value int
}

func (n *Number) Interpret() int {
    return n.value
}

// Add は加算（非終端式）
type Add struct {
    left, right Expression
}

func (a *Add) Interpret() int {
    return a.left.Interpret() + a.right.Interpret()
}

// Subtract は減算（非終端式）
type Subtract struct {
    left, right Expression
}

func (s *Subtract) Interpret() int {
    return s.left.Interpret() - s.right.Interpret()
}

// Multiply は乗算（非終端式）
type Multiply struct {
    left, right Expression
}

func (m *Multiply) Interpret() int {
    return m.left.Interpret() * m.right.Interpret()
}

func main() {
    // (5 + 3) * 2 - 4
    expr := &Subtract{
        left: &Multiply{
            left: &Add{
                left:  &Number{value: 5},
                right: &Number{value: 3},
            },
            right: &Number{value: 2},
        },
        right: &Number{value: 4},
    }

    fmt.Printf("(5 + 3) * 2 - 4 = %d\n", expr.Interpret())
}
```

### ブール式の評価

```go
package main

import "fmt"

// BoolExpression はブール式のインターフェース
type BoolExpression interface {
    Interpret(context map[string]bool) bool
}

// Variable は変数（終端式）
type Variable struct {
    name string
}

func (v *Variable) Interpret(ctx map[string]bool) bool {
    return ctx[v.name]
}

// And はAND演算（非終端式）
type And struct {
    left, right BoolExpression
}

func (a *And) Interpret(ctx map[string]bool) bool {
    return a.left.Interpret(ctx) && a.right.Interpret(ctx)
}

// Or はOR演算（非終端式）
type Or struct {
    left, right BoolExpression
}

func (o *Or) Interpret(ctx map[string]bool) bool {
    return o.left.Interpret(ctx) || o.right.Interpret(ctx)
}

// Not はNOT演算（非終端式）
type Not struct {
    expr BoolExpression
}

func (n *Not) Interpret(ctx map[string]bool) bool {
    return !n.expr.Interpret(ctx)
}

func main() {
    // (A AND B) OR (NOT C)
    expr := &Or{
        left: &And{
            left:  &Variable{name: "A"},
            right: &Variable{name: "B"},
        },
        right: &Not{
            expr: &Variable{name: "C"},
        },
    }

    contexts := []map[string]bool{
        {"A": true, "B": true, "C": false},
        {"A": true, "B": false, "C": false},
        {"A": false, "B": false, "C": true},
    }

    for _, ctx := range contexts {
        result := expr.Interpret(ctx)
        fmt.Printf("A=%v, B=%v, C=%v => (A AND B) OR (NOT C) = %v\n",
            ctx["A"], ctx["B"], ctx["C"], result)
    }
}
```

### SQLライクなクエリ

```go
package main

import (
    "fmt"
    "strings"
)

type Record map[string]string

type Condition interface {
    Match(record Record) bool
}

// Equals は等価条件
type Equals struct {
    field, value string
}

func (e *Equals) Match(r Record) bool {
    return r[e.field] == e.value
}

// Contains は部分一致条件
type Contains struct {
    field, value string
}

func (c *Contains) Match(r Record) bool {
    return strings.Contains(r[c.field], c.value)
}

// AndCondition はAND条件
type AndCondition struct {
    conditions []Condition
}

func (a *AndCondition) Match(r Record) bool {
    for _, c := range a.conditions {
        if !c.Match(r) {
            return false
        }
    }
    return true
}

// OrCondition はOR条件
type OrCondition struct {
    conditions []Condition
}

func (o *OrCondition) Match(r Record) bool {
    for _, c := range o.conditions {
        if c.Match(r) {
            return true
        }
    }
    return false
}

func Query(records []Record, condition Condition) []Record {
    var results []Record
    for _, r := range records {
        if condition.Match(r) {
            results = append(results, r)
        }
    }
    return results
}

func main() {
    records := []Record{
        {"name": "田中太郎", "dept": "開発", "city": "東京"},
        {"name": "山田花子", "dept": "営業", "city": "大阪"},
        {"name": "佐藤次郎", "dept": "開発", "city": "東京"},
        {"name": "鈴木三郎", "dept": "人事", "city": "東京"},
    }

    // dept = "開発" AND city = "東京"
    query := &AndCondition{
        conditions: []Condition{
            &Equals{field: "dept", value: "開発"},
            &Equals{field: "city", value: "東京"},
        },
    }

    results := Query(records, query)
    fmt.Println("開発部 AND 東京:")
    for _, r := range results {
        fmt.Printf("  %v\n", r)
    }
}
```

## Goでのポイント

### text/templateの活用

Go標準ライブラリの`text/template`はInterpreterパターンの実装例です：

```go
tmpl, _ := template.New("test").Parse("Hello, {{.Name}}!")
tmpl.Execute(os.Stdout, map[string]string{"Name": "World"})
```

## まとめ

- Interpreterは言語の文法を定義し解釈する
- DSL、ルールエンジン、式評価に活用
- 複雑な文法には、専用のパーサーライブラリを検討

次章では、コレクションを順次走査するIteratorパターンを学びます。
