---
title: "Proxy パターン"
---

# Proxy パターン

## 概要

Proxyパターンは、別のオブジェクトへのアクセスを制御する代理オブジェクトを提供するパターンです。元のオブジェクトと同じインターフェースを持ち、クライアントからは透過的に使用できます。

## 問題

以下のような状況で使用を検討します：

- オブジェクトへのアクセスを制御したい（認証・認可）
- 重いオブジェクトの生成を遅延させたい（遅延初期化）
- リモートオブジェクトへのアクセスを隠蔽したい
- 追加の処理（ロギング、キャッシュ）を挟みたい

## Proxyの種類

| 種類 | 目的 |
|------|------|
| Virtual Proxy | 重いオブジェクトの遅延生成 |
| Protection Proxy | アクセス制御（認証・認可） |
| Remote Proxy | リモートオブジェクトへのアクセス |
| Logging Proxy | ロギング機能の追加 |
| Caching Proxy | キャッシュ機能の追加 |

## 構造

```
┌─────────────────┐
│   Subject       │
│  (interface)    │
├─────────────────┤
│ + Request()     │
└─────────────────┘
        △
        │
   ┌────┴────┐
   │         │
┌──────┐  ┌──────────┐
│Real  │  │  Proxy   │
│Subj. │◀─│- realSub.│
└──────┘  └──────────┘
```

## 実装

### Virtual Proxy（遅延初期化）

```go
package main

import (
    "fmt"
    "time"
)

// Image はイメージを表すインターフェース
type Image interface {
    Display()
    GetFilename() string
}

// RealImage は実際のイメージ（読み込みに時間がかかる）
type RealImage struct {
    filename string
    data     []byte
}

func NewRealImage(filename string) *RealImage {
    img := &RealImage{filename: filename}
    img.loadFromDisk()
    return img
}

func (r *RealImage) loadFromDisk() {
    fmt.Printf("Loading image from disk: %s\n", r.filename)
    time.Sleep(2 * time.Second) // 重い処理をシミュレート
    r.data = []byte("image data")
    fmt.Printf("Image loaded: %s\n", r.filename)
}

func (r *RealImage) Display() {
    fmt.Printf("Displaying image: %s\n", r.filename)
}

func (r *RealImage) GetFilename() string {
    return r.filename
}

// ImageProxy は遅延読み込みを行うプロキシ
type ImageProxy struct {
    filename  string
    realImage *RealImage
}

func NewImageProxy(filename string) *ImageProxy {
    return &ImageProxy{filename: filename}
}

func (p *ImageProxy) Display() {
    if p.realImage == nil {
        p.realImage = NewRealImage(p.filename)
    }
    p.realImage.Display()
}

func (p *ImageProxy) GetFilename() string {
    return p.filename
}

func main() {
    fmt.Println("=== Creating proxies (fast) ===")
    images := []Image{
        NewImageProxy("photo1.jpg"),
        NewImageProxy("photo2.jpg"),
        NewImageProxy("photo3.jpg"),
    }

    fmt.Println("\n=== Displaying first image (loads now) ===")
    images[0].Display()

    fmt.Println("\n=== Displaying first image again (already loaded) ===")
    images[0].Display()

    fmt.Println("\n=== Other images are not loaded yet ===")
    fmt.Printf("Image 2 filename: %s\n", images[1].GetFilename())
}
```

### Protection Proxy（アクセス制御）

```go
package main

import (
    "errors"
    "fmt"
)

// Document はドキュメントを表すインターフェース
type Document interface {
    Read() string
    Write(content string) error
    Delete() error
}

// RealDocument は実際のドキュメント
type RealDocument struct {
    content string
}

func (d *RealDocument) Read() string {
    return d.content
}

func (d *RealDocument) Write(content string) error {
    d.content = content
    fmt.Println("Document written")
    return nil
}

func (d *RealDocument) Delete() error {
    d.content = ""
    fmt.Println("Document deleted")
    return nil
}

// User はユーザー情報
type User struct {
    Name string
    Role string // "admin", "editor", "viewer"
}

// ProtectedDocument はアクセス制御を行うプロキシ
type ProtectedDocument struct {
    document *RealDocument
    user     *User
}

func NewProtectedDocument(doc *RealDocument, user *User) *ProtectedDocument {
    return &ProtectedDocument{document: doc, user: user}
}

func (p *ProtectedDocument) Read() string {
    // 誰でも読める
    fmt.Printf("[%s] Reading document\n", p.user.Name)
    return p.document.Read()
}

func (p *ProtectedDocument) Write(content string) error {
    if p.user.Role != "admin" && p.user.Role != "editor" {
        return errors.New("permission denied: write requires admin or editor role")
    }
    fmt.Printf("[%s] Writing document\n", p.user.Name)
    return p.document.Write(content)
}

func (p *ProtectedDocument) Delete() error {
    if p.user.Role != "admin" {
        return errors.New("permission denied: delete requires admin role")
    }
    fmt.Printf("[%s] Deleting document\n", p.user.Name)
    return p.document.Delete()
}

func main() {
    doc := &RealDocument{content: "Original content"}

    admin := &User{Name: "Admin User", Role: "admin"}
    editor := &User{Name: "Editor User", Role: "editor"}
    viewer := &User{Name: "Viewer User", Role: "viewer"}

    fmt.Println("=== Admin access ===")
    adminDoc := NewProtectedDocument(doc, admin)
    fmt.Println(adminDoc.Read())
    adminDoc.Write("Updated by admin")
    adminDoc.Delete()

    fmt.Println("\n=== Editor access ===")
    editorDoc := NewProtectedDocument(doc, editor)
    editorDoc.Write("New content by editor")
    if err := editorDoc.Delete(); err != nil {
        fmt.Printf("Error: %v\n", err)
    }

    fmt.Println("\n=== Viewer access ===")
    viewerDoc := NewProtectedDocument(doc, viewer)
    fmt.Println(viewerDoc.Read())
    if err := viewerDoc.Write("Trying to write"); err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

### Caching Proxy

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// DataFetcher はデータ取得のインターフェース
type DataFetcher interface {
    Fetch(key string) (string, error)
}

// SlowDataFetcher は遅いデータソース
type SlowDataFetcher struct{}

func (s *SlowDataFetcher) Fetch(key string) (string, error) {
    fmt.Printf("Fetching data for key: %s (slow operation)\n", key)
    time.Sleep(1 * time.Second)
    return fmt.Sprintf("Data for %s", key), nil
}

// CachingProxy はキャッシュ機能を持つプロキシ
type CachingProxy struct {
    fetcher DataFetcher
    cache   map[string]cacheEntry
    ttl     time.Duration
    mu      sync.RWMutex
}

type cacheEntry struct {
    value     string
    expiresAt time.Time
}

func NewCachingProxy(fetcher DataFetcher, ttl time.Duration) *CachingProxy {
    return &CachingProxy{
        fetcher: fetcher,
        cache:   make(map[string]cacheEntry),
        ttl:     ttl,
    }
}

func (c *CachingProxy) Fetch(key string) (string, error) {
    c.mu.RLock()
    if entry, ok := c.cache[key]; ok && time.Now().Before(entry.expiresAt) {
        c.mu.RUnlock()
        fmt.Printf("[Cache Hit] %s\n", key)
        return entry.value, nil
    }
    c.mu.RUnlock()

    // キャッシュミス - 実際にフェッチ
    fmt.Printf("[Cache Miss] %s\n", key)
    value, err := c.fetcher.Fetch(key)
    if err != nil {
        return "", err
    }

    c.mu.Lock()
    c.cache[key] = cacheEntry{
        value:     value,
        expiresAt: time.Now().Add(c.ttl),
    }
    c.mu.Unlock()

    return value, nil
}

func main() {
    slowFetcher := &SlowDataFetcher{}
    cachedFetcher := NewCachingProxy(slowFetcher, 5*time.Second)

    fmt.Println("=== First fetch (slow) ===")
    start := time.Now()
    data, _ := cachedFetcher.Fetch("user:123")
    fmt.Printf("Result: %s (took %v)\n\n", data, time.Since(start))

    fmt.Println("=== Second fetch (cached) ===")
    start = time.Now()
    data, _ = cachedFetcher.Fetch("user:123")
    fmt.Printf("Result: %s (took %v)\n\n", data, time.Since(start))

    fmt.Println("=== Different key (slow) ===")
    start = time.Now()
    data, _ = cachedFetcher.Fetch("user:456")
    fmt.Printf("Result: %s (took %v)\n", data, time.Since(start))
}
```

### Logging Proxy

```go
package main

import (
    "fmt"
    "log"
    "time"
)

// Calculator は計算機のインターフェース
type Calculator interface {
    Add(a, b int) int
    Multiply(a, b int) int
    Divide(a, b int) (int, error)
}

// RealCalculator は実際の計算機
type RealCalculator struct{}

func (c *RealCalculator) Add(a, b int) int {
    return a + b
}

func (c *RealCalculator) Multiply(a, b int) int {
    return a * b
}

func (c *RealCalculator) Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

// LoggingProxy はロギング機能を持つプロキシ
type LoggingProxy struct {
    calculator Calculator
}

func NewLoggingProxy(calc Calculator) *LoggingProxy {
    return &LoggingProxy{calculator: calc}
}

func (p *LoggingProxy) Add(a, b int) int {
    start := time.Now()
    result := p.calculator.Add(a, b)
    log.Printf("Add(%d, %d) = %d [%v]", a, b, result, time.Since(start))
    return result
}

func (p *LoggingProxy) Multiply(a, b int) int {
    start := time.Now()
    result := p.calculator.Multiply(a, b)
    log.Printf("Multiply(%d, %d) = %d [%v]", a, b, result, time.Since(start))
    return result
}

func (p *LoggingProxy) Divide(a, b int) (int, error) {
    start := time.Now()
    result, err := p.calculator.Divide(a, b)
    if err != nil {
        log.Printf("Divide(%d, %d) = error: %v [%v]", a, b, err, time.Since(start))
    } else {
        log.Printf("Divide(%d, %d) = %d [%v]", a, b, result, time.Since(start))
    }
    return result, err
}

func main() {
    calc := NewLoggingProxy(&RealCalculator{})

    calc.Add(10, 5)
    calc.Multiply(6, 7)
    calc.Divide(20, 4)
    calc.Divide(10, 0)
}
```

## 実践的な例：HTTPクライアントプロキシ

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// HTTPClient はHTTPクライアントのインターフェース
type HTTPClient interface {
    Get(url string) (string, error)
}

// RealHTTPClient は実際のHTTPクライアント
type RealHTTPClient struct{}

func (c *RealHTTPClient) Get(url string) (string, error) {
    fmt.Printf("Fetching: %s\n", url)
    time.Sleep(500 * time.Millisecond) // ネットワーク遅延
    return fmt.Sprintf("Response from %s", url), nil
}

// RateLimitingProxy はレート制限付きプロキシ
type RateLimitingProxy struct {
    client      HTTPClient
    requests    int
    maxRequests int
    window      time.Duration
    windowStart time.Time
    mu          sync.Mutex
}

func NewRateLimitingProxy(client HTTPClient, maxReq int, window time.Duration) *RateLimitingProxy {
    return &RateLimitingProxy{
        client:      client,
        maxRequests: maxReq,
        window:      window,
        windowStart: time.Now(),
    }
}

func (p *RateLimitingProxy) Get(url string) (string, error) {
    p.mu.Lock()
    defer p.mu.Unlock()

    // ウィンドウをリセット
    if time.Since(p.windowStart) > p.window {
        p.requests = 0
        p.windowStart = time.Now()
    }

    if p.requests >= p.maxRequests {
        return "", fmt.Errorf("rate limit exceeded: max %d requests per %v",
            p.maxRequests, p.window)
    }

    p.requests++
    fmt.Printf("[RateLimit] Request %d/%d\n", p.requests, p.maxRequests)
    return p.client.Get(url)
}

// RetryProxy は自動リトライ機能付きプロキシ
type RetryProxy struct {
    client     HTTPClient
    maxRetries int
}

func NewRetryProxy(client HTTPClient, maxRetries int) *RetryProxy {
    return &RetryProxy{client: client, maxRetries: maxRetries}
}

func (p *RetryProxy) Get(url string) (string, error) {
    var lastErr error
    for i := 0; i <= p.maxRetries; i++ {
        if i > 0 {
            fmt.Printf("[Retry] Attempt %d/%d\n", i+1, p.maxRetries+1)
            time.Sleep(time.Duration(i) * 100 * time.Millisecond)
        }
        result, err := p.client.Get(url)
        if err == nil {
            return result, nil
        }
        lastErr = err
    }
    return "", fmt.Errorf("all retries failed: %w", lastErr)
}

func main() {
    client := &RealHTTPClient{}

    // レート制限プロキシをラップ
    rateLimited := NewRateLimitingProxy(client, 3, 5*time.Second)

    fmt.Println("=== Rate Limited Requests ===")
    for i := 0; i < 5; i++ {
        result, err := rateLimited.Get(fmt.Sprintf("https://api.example.com/data/%d", i))
        if err != nil {
            fmt.Printf("Error: %v\n", err)
        } else {
            fmt.Println(result)
        }
    }
}
```

## Goでのポイント

### インターフェースによる透過性

```go
// クライアントコードはインターフェースのみを使用
func ProcessData(client HTTPClient) {
    // Proxyか実体かを気にしない
    client.Get("https://api.example.com")
}
```

### Proxyのチェーン

複数のProxyを組み合わせられます：

```go
client := &RealHTTPClient{}
logged := NewLoggingProxy(client)
cached := NewCachingProxy(logged)
rateLimited := NewRateLimitingProxy(cached)
```

## Decoratorとの違い

| 観点 | Proxy | Decorator |
|-----|-------|-----------|
| 目的 | アクセス制御 | 機能追加 |
| 対象管理 | Proxyが対象のライフサイクルを管理することが多い | クライアントが対象を渡す |
| 用途 | 遅延初期化、認可、キャッシュ | 追加の振る舞い |

## まとめ

- Proxyはオブジェクトへのアクセスを制御する代理オブジェクト
- 遅延初期化、認可、キャッシュ、ロギングなど多様な用途
- Goではインターフェースを使って透過的に実装できる

これで構造に関するパターンの解説は終わりです。次章からは、振る舞いに関するパターンを学びます。
