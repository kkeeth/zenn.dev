---
title: "Decorator パターン"
---

# Decorator パターン

## 概要

Decoratorパターンは、オブジェクトに動的に新しい責務（機能）を追加するパターンです。継承よりも柔軟な機能拡張を提供します。

## 問題

以下のような状況で使用を検討します：

- 実行時に動的に機能を追加・削除したい
- 継承による機能拡張が爆発的に増える（クラス爆発問題）
- 元のオブジェクトを変更せずに機能を追加したい

## 構造

```
┌─────────────────┐
│   Component     │
│  (interface)    │
├─────────────────┤
│ + Operation()   │
└─────────────────┘
        △
        │
   ┌────┴────────┐
   │             │
┌──────┐  ┌────────────┐
│Concre│  │ Decorator  │───┐
│te    │  ├────────────┤   │
└──────┘  │- component │◀──┘
          └────────────┘
```

## 実装

### コーヒーショップの例

```go
package main

import "fmt"

// Beverage は飲み物を表すインターフェース
type Beverage interface {
    GetDescription() string
    Cost() float64
}

// ===== 具象コンポーネント =====

// Espresso はエスプレッソ
type Espresso struct{}

func (e *Espresso) GetDescription() string { return "エスプレッソ" }
func (e *Espresso) Cost() float64          { return 300 }

// HouseBlend はハウスブレンド
type HouseBlend struct{}

func (h *HouseBlend) GetDescription() string { return "ハウスブレンド" }
func (h *HouseBlend) Cost() float64          { return 250 }

// ===== デコレータ =====

// Milk はミルクを追加するデコレータ
type Milk struct {
    beverage Beverage
}

func WithMilk(b Beverage) *Milk {
    return &Milk{beverage: b}
}

func (m *Milk) GetDescription() string {
    return m.beverage.GetDescription() + " + ミルク"
}

func (m *Milk) Cost() float64 {
    return m.beverage.Cost() + 50
}

// Mocha はモカ（チョコレート）を追加するデコレータ
type Mocha struct {
    beverage Beverage
}

func WithMocha(b Beverage) *Mocha {
    return &Mocha{beverage: b}
}

func (m *Mocha) GetDescription() string {
    return m.beverage.GetDescription() + " + モカ"
}

func (m *Mocha) Cost() float64 {
    return m.beverage.Cost() + 80
}

// WhippedCream はホイップクリームを追加するデコレータ
type WhippedCream struct {
    beverage Beverage
}

func WithWhippedCream(b Beverage) *WhippedCream {
    return &WhippedCream{beverage: b}
}

func (w *WhippedCream) GetDescription() string {
    return w.beverage.GetDescription() + " + ホイップ"
}

func (w *WhippedCream) Cost() float64 {
    return w.beverage.Cost() + 70
}

func main() {
    // シンプルなエスプレッソ
    espresso := &Espresso{}
    fmt.Printf("%s: ¥%.0f\n", espresso.GetDescription(), espresso.Cost())

    // カスタマイズしたドリンク
    var drink Beverage = &HouseBlend{}
    drink = WithMilk(drink)
    drink = WithMocha(drink)
    drink = WithMocha(drink) // ダブルモカ
    drink = WithWhippedCream(drink)

    fmt.Printf("%s: ¥%.0f\n", drink.GetDescription(), drink.Cost())
}
```

### データストリームの例

```go
package main

import (
    "compress/gzip"
    "encoding/base64"
    "fmt"
    "io"
    "strings"
)

// DataSource はデータソースのインターフェース
type DataSource interface {
    WriteData(data string) error
    ReadData() (string, error)
}

// ===== 具象コンポーネント =====

// FileDataSource はファイルデータソース
type FileDataSource struct {
    filename string
    data     string // 簡略化のためメモリに保存
}

func NewFileDataSource(filename string) *FileDataSource {
    return &FileDataSource{filename: filename}
}

func (f *FileDataSource) WriteData(data string) error {
    fmt.Printf("[File] Writing to %s\n", f.filename)
    f.data = data
    return nil
}

func (f *FileDataSource) ReadData() (string, error) {
    fmt.Printf("[File] Reading from %s\n", f.filename)
    return f.data, nil
}

// ===== デコレータの基底 =====

// DataSourceDecorator はデコレータの基底
type DataSourceDecorator struct {
    wrappee DataSource
}

// ===== 具象デコレータ =====

// EncryptionDecorator は暗号化デコレータ
type EncryptionDecorator struct {
    DataSourceDecorator
}

func WithEncryption(source DataSource) *EncryptionDecorator {
    return &EncryptionDecorator{
        DataSourceDecorator: DataSourceDecorator{wrappee: source},
    }
}

func (e *EncryptionDecorator) WriteData(data string) error {
    // 簡略化: Base64エンコードを暗号化の代わりに使用
    encoded := base64.StdEncoding.EncodeToString([]byte(data))
    fmt.Println("[Encrypt] Encrypting data")
    return e.wrappee.WriteData(encoded)
}

func (e *EncryptionDecorator) ReadData() (string, error) {
    data, err := e.wrappee.ReadData()
    if err != nil {
        return "", err
    }
    decoded, err := base64.StdEncoding.DecodeString(data)
    if err != nil {
        return "", err
    }
    fmt.Println("[Encrypt] Decrypting data")
    return string(decoded), nil
}

// CompressionDecorator は圧縮デコレータ
type CompressionDecorator struct {
    DataSourceDecorator
}

func WithCompression(source DataSource) *CompressionDecorator {
    return &CompressionDecorator{
        DataSourceDecorator: DataSourceDecorator{wrappee: source},
    }
}

func (c *CompressionDecorator) WriteData(data string) error {
    var buf strings.Builder
    w := gzip.NewWriter(&buf)
    w.Write([]byte(data))
    w.Close()
    fmt.Println("[Compress] Compressing data")
    return c.wrappee.WriteData(buf.String())
}

func (c *CompressionDecorator) ReadData() (string, error) {
    data, err := c.wrappee.ReadData()
    if err != nil {
        return "", err
    }
    r, err := gzip.NewReader(strings.NewReader(data))
    if err != nil {
        return "", err
    }
    defer r.Close()
    decompressed, _ := io.ReadAll(r)
    fmt.Println("[Compress] Decompressing data")
    return string(decompressed), nil
}

func main() {
    // シンプルなファイル操作
    fmt.Println("=== Simple File ===")
    simple := NewFileDataSource("data.txt")
    simple.WriteData("Hello, World!")
    data, _ := simple.ReadData()
    fmt.Printf("Data: %s\n\n", data)

    // 暗号化付きファイル
    fmt.Println("=== Encrypted File ===")
    encrypted := WithEncryption(NewFileDataSource("encrypted.txt"))
    encrypted.WriteData("Secret Message")
    data, _ = encrypted.ReadData()
    fmt.Printf("Data: %s\n\n", data)

    // 圧縮 + 暗号化
    fmt.Println("=== Compressed + Encrypted File ===")
    both := WithEncryption(WithCompression(NewFileDataSource("both.txt")))
    both.WriteData("Compressed and Encrypted Data")
    data, _ = both.ReadData()
    fmt.Printf("Data: %s\n", data)
}
```

## 実践的な例：HTTPミドルウェア

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// ===== ミドルウェア（Decorator）=====

// LoggingMiddleware はロギングミドルウェア
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        log.Printf("[%s] %s - Started", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
        log.Printf("[%s] %s - Completed in %v", r.Method, r.URL.Path, time.Since(start))
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
        log.Printf("[Auth] Token validated")
        next.ServeHTTP(w, r)
    })
}

// RecoveryMiddleware はパニック回復ミドルウェア
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("[Recovery] Panic recovered: %v", err)
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// ===== ハンドラー =====

func HelloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    // ハンドラーをデコレート（ミドルウェアを積み重ね）
    handler := http.HandlerFunc(HelloHandler)

    // ミドルウェアを適用（内側から外側の順）
    decorated := RecoveryMiddleware(
        LoggingMiddleware(
            AuthMiddleware(handler),
        ),
    )

    fmt.Println("Server starting on :8080")
    fmt.Println("Try: curl -H 'Authorization: token123' http://localhost:8080")
    // http.ListenAndServe(":8080", decorated)
    _ = decorated // デモ用
}
```

## 実践的な例：関数デコレータ

```go
package main

import (
    "fmt"
    "time"
)

// ===== 関数型デコレータ =====

// Operation は処理関数の型
type Operation func(x, y int) int

// WithLogging はロギングを追加
func WithLogging(op Operation, name string) Operation {
    return func(x, y int) int {
        fmt.Printf("[Log] Calling %s(%d, %d)\n", name, x, y)
        result := op(x, y)
        fmt.Printf("[Log] Result: %d\n", result)
        return result
    }
}

// WithTiming は実行時間計測を追加
func WithTiming(op Operation) Operation {
    return func(x, y int) int {
        start := time.Now()
        result := op(x, y)
        fmt.Printf("[Time] Elapsed: %v\n", time.Since(start))
        return result
    }
}

// WithCache はキャッシュを追加
func WithCache(op Operation) Operation {
    cache := make(map[string]int)
    return func(x, y int) int {
        key := fmt.Sprintf("%d:%d", x, y)
        if cached, ok := cache[key]; ok {
            fmt.Println("[Cache] Hit!")
            return cached
        }
        result := op(x, y)
        cache[key] = result
        fmt.Println("[Cache] Miss, cached")
        return result
    }
}

// ===== 基本関数 =====

func Add(x, y int) int {
    return x + y
}

func SlowMultiply(x, y int) int {
    time.Sleep(100 * time.Millisecond) // 重い処理をシミュレート
    return x * y
}

func main() {
    // ロギング付きの加算
    loggedAdd := WithLogging(Add, "Add")
    loggedAdd(3, 5)

    fmt.Println()

    // タイミング + キャッシュ付きの乗算
    cachedMultiply := WithCache(WithTiming(SlowMultiply))

    fmt.Println("First call:")
    cachedMultiply(4, 5)

    fmt.Println("\nSecond call (cached):")
    cachedMultiply(4, 5)
}
```

## Goでのポイント

### インターフェースを満たすことが重要

デコレータは元のインターフェースと同じインターフェースを実装します：

```go
type Component interface {
    Operation()
}

type Decorator struct {
    component Component // 同じインターフェース型を持つ
}

// Decoratorも同じインターフェースを実装
func (d *Decorator) Operation() {
    // 前処理
    d.component.Operation()
    // 後処理
}
```

### io.Readerのデコレータ

標準ライブラリにもデコレータパターンの例があります：

```go
// ファイルを読む
file, _ := os.Open("file.txt")

// バッファリングを追加
buffered := bufio.NewReader(file)

// さらに制限を追加
limited := io.LimitReader(buffered, 1024)
```

## Compositeとの違い

| 観点 | Decorator | Composite |
|-----|-----------|-----------|
| 目的 | 機能の追加 | 階層構造の表現 |
| 子の数 | 常に1つ | 複数可能 |
| 構造 | 線形（チェーン） | 木構造 |

## まとめ

- Decoratorはオブジェクトに動的に機能を追加する
- 継承より柔軟で、組み合わせ自由
- GoではHTTPミドルウェア、io.Readerなどで広く使われている

次章では、複雑なサブシステムへのシンプルなインターフェースを提供するFacadeパターンを学びます。
