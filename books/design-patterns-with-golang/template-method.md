---
title: "Template Method パターン"
---

# Template Method パターン

## 概要

Template Methodパターンは、アルゴリズムの骨格を定義し、一部のステップをサブクラス（または関数）に委譲するパターンです。アルゴリズムの構造を変えずに、特定のステップをオーバーライドできます。

## 問題

以下のような状況で使用を検討します：

- アルゴリズムの骨格は共通だが、一部のステップが異なる
- コードの重複を避けたい
- 拡張ポイントを明確に定義したい

## 構造

```
┌───────────────────────┐
│   AbstractClass       │
├───────────────────────┤
│ + TemplateMethod()    │  ←アルゴリズムの骨格
│ # Step1()             │  ←抽象メソッド
│ # Step2()             │
│ - FixedStep()         │  ←固定のステップ
└───────────────────────┘
           △
      ┌────┴────┐
┌─────────┐  ┌─────────┐
│ ClassA  │  │ ClassB  │
│ Step1() │  │ Step1() │
│ Step2() │  │ Step2() │
└─────────┘  └─────────┘
```

## 実装

### 埋め込みを使った実装

```go
package main

import "fmt"

// DataMiner はデータマイニングのステップを定義
type DataMiner interface {
    OpenFile(path string) error
    ExtractData() string
    ParseData(data string) []string
    Analyze(data []string)
    CloseFile()
}

// BaseMiner は共通のテンプレートメソッドを持つ
type BaseMiner struct {
    impl DataMiner
}

// Mine はテンプレートメソッド（アルゴリズムの骨格）
func (b *BaseMiner) Mine(path string) {
    fmt.Println("=== データマイニング開始 ===")
    b.impl.OpenFile(path)
    rawData := b.impl.ExtractData()
    parsedData := b.impl.ParseData(rawData)
    b.impl.Analyze(parsedData)
    b.impl.CloseFile()
    fmt.Println("=== データマイニング完了 ===")
}

// CSVMiner はCSVファイル用のマイナー
type CSVMiner struct {
    BaseMiner
}

func NewCSVMiner() *CSVMiner {
    m := &CSVMiner{}
    m.impl = m
    return m
}

func (m *CSVMiner) OpenFile(path string) error {
    fmt.Printf("CSVファイルを開く: %s\n", path)
    return nil
}

func (m *CSVMiner) ExtractData() string {
    return "name,age\nAlice,30\nBob,25"
}

func (m *CSVMiner) ParseData(data string) []string {
    fmt.Println("CSVをパース中...")
    return []string{"Alice:30", "Bob:25"}
}

func (m *CSVMiner) Analyze(data []string) {
    fmt.Printf("分析結果: %d 件のレコード\n", len(data))
}

func (m *CSVMiner) CloseFile() {
    fmt.Println("CSVファイルを閉じる")
}

// JSONMiner はJSONファイル用のマイナー
type JSONMiner struct {
    BaseMiner
}

func NewJSONMiner() *JSONMiner {
    m := &JSONMiner{}
    m.impl = m
    return m
}

func (m *JSONMiner) OpenFile(path string) error {
    fmt.Printf("JSONファイルを開く: %s\n", path)
    return nil
}

func (m *JSONMiner) ExtractData() string {
    return `[{"name":"Alice"},{"name":"Bob"}]`
}

func (m *JSONMiner) ParseData(data string) []string {
    fmt.Println("JSONをパース中...")
    return []string{"Alice", "Bob"}
}

func (m *JSONMiner) Analyze(data []string) {
    fmt.Printf("分析結果: %d 件のオブジェクト\n", len(data))
}

func (m *JSONMiner) CloseFile() {
    fmt.Println("JSONファイルを閉じる")
}

func main() {
    csvMiner := NewCSVMiner()
    csvMiner.Mine("data.csv")

    fmt.Println()

    jsonMiner := NewJSONMiner()
    jsonMiner.Mine("data.json")
}
```

### 関数型での実装（Goでシンプル）

```go
package main

import "fmt"

// ReportTemplate はレポート生成のテンプレート
type ReportTemplate struct {
    FetchData   func() []string
    FormatData  func([]string) string
    GenerateReport func(string) string
}

// Generate はテンプレートメソッド
func (t *ReportTemplate) Generate() string {
    fmt.Println("1. データ取得中...")
    data := t.FetchData()

    fmt.Println("2. データフォーマット中...")
    formatted := t.FormatData(data)

    fmt.Println("3. レポート生成中...")
    return t.GenerateReport(formatted)
}

func main() {
    // HTMLレポート
    htmlReport := &ReportTemplate{
        FetchData: func() []string {
            return []string{"Item A", "Item B", "Item C"}
        },
        FormatData: func(data []string) string {
            result := "<ul>"
            for _, item := range data {
                result += "<li>" + item + "</li>"
            }
            return result + "</ul>"
        },
        GenerateReport: func(content string) string {
            return "<html><body>" + content + "</body></html>"
        },
    }

    fmt.Println("=== HTMLレポート ===")
    fmt.Println(htmlReport.Generate())

    // Markdownレポート
    mdReport := &ReportTemplate{
        FetchData: func() []string {
            return []string{"Item A", "Item B", "Item C"}
        },
        FormatData: func(data []string) string {
            result := ""
            for _, item := range data {
                result += "- " + item + "\n"
            }
            return result
        },
        GenerateReport: func(content string) string {
            return "# Report\n\n" + content
        },
    }

    fmt.Println("\n=== Markdownレポート ===")
    fmt.Println(mdReport.Generate())
}
```

### HTTPハンドラの例

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Request struct {
    Body map[string]any
}

type Response struct {
    Status int
    Body   any
}

// HandlerTemplate はHTTPハンドラのテンプレート
type HandlerTemplate struct {
    Validate func(Request) error
    Process  func(Request) (any, error)
    Format   func(any) Response
}

func (h *HandlerTemplate) Handle(req Request) Response {
    // 1. バリデーション
    if err := h.Validate(req); err != nil {
        return Response{Status: 400, Body: err.Error()}
    }

    // 2. 処理
    result, err := h.Process(req)
    if err != nil {
        return Response{Status: 500, Body: err.Error()}
    }

    // 3. レスポンス生成
    return h.Format(result)
}

func main() {
    userHandler := &HandlerTemplate{
        Validate: func(req Request) error {
            if req.Body["name"] == nil {
                return fmt.Errorf("name is required")
            }
            return nil
        },
        Process: func(req Request) (any, error) {
            return map[string]any{
                "id":   1,
                "name": req.Body["name"],
            }, nil
        },
        Format: func(result any) Response {
            return Response{Status: 200, Body: result}
        },
    }

    req := Request{Body: map[string]any{"name": "田中"}}
    resp := userHandler.Handle(req)
    data, _ := json.Marshal(resp)
    fmt.Println(string(data))
}
```

## Goでのポイント

### hookパターン

特定のポイントにフックを挿入する：

```go
type Processor struct {
    BeforeProcess func()
    AfterProcess  func()
}

func (p *Processor) Process() {
    if p.BeforeProcess != nil {
        p.BeforeProcess()
    }
    // メインの処理
    if p.AfterProcess != nil {
        p.AfterProcess()
    }
}
```

## Strategyとの違い

| 観点 | Template Method | Strategy |
|-----|----------------|----------|
| 変更単位 | アルゴリズムの一部 | アルゴリズム全体 |
| 継承/委譲 | 継承（埋め込み） | 委譲（コンポジション） |
| 制御 | 親が骨格を制御 | クライアントが戦略を選択 |

## まとめ

- Template Methodはアルゴリズムの骨格を定義し、一部を委譲
- Goでは埋め込みと関数型で実装できる
- コードの重複を減らし、拡張ポイントを明確にする

次章では、構造を変えずに新しい操作を追加するVisitorパターンを学びます。
