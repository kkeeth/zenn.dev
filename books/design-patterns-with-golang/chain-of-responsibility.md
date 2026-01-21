---
title: "Chain of Responsibility パターン"
---

# Chain of Responsibility パターン

## 概要

Chain of Responsibilityパターンは、リクエストを処理するオブジェクトの連鎖を構築し、リクエストを受け取ったオブジェクトが処理するか、次のオブジェクトに渡すかを決定するパターンです。

## 問題

以下のような状況で使用を検討します：

- リクエストの送信者と受信者を分離したい
- 複数のオブジェクトがリクエストを処理する可能性がある
- 処理者を動的に変更したい

## 構造

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Handler A  │────▶│  Handler B  │────▶│  Handler C  │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ + Handle()  │     │ + Handle()  │     │ + Handle()  │
└─────────────┘     └─────────────┘     └─────────────┘
```

## 実装

### 基本的な実装

```go
package main

import "fmt"

// Handler はハンドラのインターフェース
type Handler interface {
    SetNext(handler Handler) Handler
    Handle(request string) string
}

// BaseHandler は共通処理を持つ基底ハンドラ
type BaseHandler struct {
    next Handler
}

func (h *BaseHandler) SetNext(handler Handler) Handler {
    h.next = handler
    return handler
}

func (h *BaseHandler) Handle(request string) string {
    if h.next != nil {
        return h.next.Handle(request)
    }
    return ""
}

// MonkeyHandler は猿が処理するハンドラ
type MonkeyHandler struct {
    BaseHandler
}

func (h *MonkeyHandler) Handle(request string) string {
    if request == "バナナ" {
        return "猿: バナナを食べます"
    }
    return h.BaseHandler.Handle(request)
}

// SquirrelHandler はリスが処理するハンドラ
type SquirrelHandler struct {
    BaseHandler
}

func (h *SquirrelHandler) Handle(request string) string {
    if request == "ナッツ" {
        return "リス: ナッツを食べます"
    }
    return h.BaseHandler.Handle(request)
}

// DogHandler は犬が処理するハンドラ
type DogHandler struct {
    BaseHandler
}

func (h *DogHandler) Handle(request string) string {
    if request == "ミートボール" {
        return "犬: ミートボールを食べます"
    }
    return h.BaseHandler.Handle(request)
}

func main() {
    monkey := &MonkeyHandler{}
    squirrel := &SquirrelHandler{}
    dog := &DogHandler{}

    // チェーンを構築
    monkey.SetNext(squirrel).SetNext(dog)

    foods := []string{"ナッツ", "バナナ", "ミートボール", "コーヒー"}

    for _, food := range foods {
        result := monkey.Handle(food)
        if result != "" {
            fmt.Println(result)
        } else {
            fmt.Printf("%s は誰も食べませんでした\n", food)
        }
    }
}
```

### HTTPミドルウェアの例（Goで最も一般的な実装）

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

// Middleware はHTTPミドルウェアの型
type Middleware func(http.Handler) http.Handler

// Chain はミドルウェアをチェーンする
func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

// LoggingMiddleware はロギングミドルウェア
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        fmt.Printf("[%s] %s - Started\n", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
        fmt.Printf("[%s] %s - Completed (%v)\n", r.Method, r.URL.Path, time.Since(start))
    })
}

// AuthMiddleware は認証ミドルウェア
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        fmt.Println("[Auth] Token validated")
        next.ServeHTTP(w, r)
    })
}

// RecoveryMiddleware はパニック回復ミドルウェア
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                fmt.Printf("[Recovery] Panic: %v\n", err)
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func HelloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    handler := Chain(
        http.HandlerFunc(HelloHandler),
        RecoveryMiddleware,
        LoggingMiddleware,
        AuthMiddleware,
    )

    // シミュレート
    fmt.Println("Server would start with chained middlewares")
    _ = handler
}
```

### 承認ワークフローの例

```go
package main

import "fmt"

type PurchaseRequest struct {
    Amount      float64
    Purpose     string
    RequesterID string
}

type Approver interface {
    SetNext(approver Approver)
    Approve(request PurchaseRequest) bool
}

type BaseApprover struct {
    next Approver
    name string
}

func (a *BaseApprover) SetNext(approver Approver) {
    a.next = approver
}

// Manager は課長（1万円まで承認可能）
type Manager struct {
    BaseApprover
}

func NewManager() *Manager {
    return &Manager{BaseApprover: BaseApprover{name: "課長"}}
}

func (m *Manager) Approve(req PurchaseRequest) bool {
    if req.Amount <= 10000 {
        fmt.Printf("%s が承認: ¥%.0f (%s)\n", m.name, req.Amount, req.Purpose)
        return true
    }
    if m.next != nil {
        return m.next.Approve(req)
    }
    return false
}

// Director は部長（10万円まで承認可能）
type Director struct {
    BaseApprover
}

func NewDirector() *Director {
    return &Director{BaseApprover: BaseApprover{name: "部長"}}
}

func (d *Director) Approve(req PurchaseRequest) bool {
    if req.Amount <= 100000 {
        fmt.Printf("%s が承認: ¥%.0f (%s)\n", d.name, req.Amount, req.Purpose)
        return true
    }
    if d.next != nil {
        return d.next.Approve(req)
    }
    return false
}

// CEO は社長（上限なし）
type CEO struct {
    BaseApprover
}

func NewCEO() *CEO {
    return &CEO{BaseApprover: BaseApprover{name: "社長"}}
}

func (c *CEO) Approve(req PurchaseRequest) bool {
    fmt.Printf("%s が承認: ¥%.0f (%s)\n", c.name, req.Amount, req.Purpose)
    return true
}

func main() {
    manager := NewManager()
    director := NewDirector()
    ceo := NewCEO()

    manager.SetNext(director)
    director.SetNext(ceo)

    requests := []PurchaseRequest{
        {Amount: 5000, Purpose: "文房具"},
        {Amount: 50000, Purpose: "ソフトウェアライセンス"},
        {Amount: 500000, Purpose: "新サーバー"},
    }

    for _, req := range requests {
        fmt.Printf("\n申請: ¥%.0f (%s)\n", req.Amount, req.Purpose)
        manager.Approve(req)
    }
}
```

## Goでのポイント

### 関数スライスによる実装

```go
type Handler func(request string) (string, bool)

func ProcessChain(request string, handlers ...Handler) string {
    for _, h := range handlers {
        if result, handled := h(request); handled {
            return result
        }
    }
    return "unhandled"
}
```

## まとめ

- Chain of Responsibilityはリクエストを処理者の連鎖に渡す
- GoではHTTPミドルウェアとして広く使われている
- 関数チェーンでシンプルに実装可能

次章では、リクエストをオブジェクトとしてカプセル化するCommandパターンを学びます。
