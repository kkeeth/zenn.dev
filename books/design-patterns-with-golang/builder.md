---
title: "Builder パターン"
---

# Builder パターン

## 概要

Builderパターンは、複雑なオブジェクトの構築をステップバイステップで行うパターンです。同じ構築プロセスで異なる表現のオブジェクトを作成できます。

## 問題

以下のような状況で使用を検討します：

- オブジェクトの生成に多くのパラメータが必要
- オプションパラメータが多い（テレスコーピングコンストラクタ問題）
- 生成されるオブジェクトは不変（immutable）にしたい
- 複雑なオブジェクトを段階的に構築したい

## テレスコーピングコンストラクタ問題

```go
// 悪い例：パラメータが増えると管理が困難
func NewUser(name string) *User
func NewUserWithAge(name string, age int) *User
func NewUserWithAgeAndEmail(name string, age int, email string) *User
func NewUserWithAgeAndEmailAndPhone(name string, age int, email, phone string) *User

// 使用時：どの引数が何を意味するか分かりにくい
user := NewUser("田中", 30, "tanaka@example.com", "090-1234-5678", true, "東京")
```

## 構造

```
┌─────────────────────┐
│      Director       │
├─────────────────────┤
│ + Construct()       │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐         ┌─────────────────────┐
│      Builder        │◁────────│   ConcreteBuilder   │
│    (interface)      │         ├─────────────────────┤
├─────────────────────┤         │ + BuildPartA()      │
│ + BuildPartA()      │         │ + BuildPartB()      │
│ + BuildPartB()      │         │ + GetResult()       │
│ + GetResult()       │         └─────────────────────┘
└─────────────────────┘
```

## 実装

### メソッドチェーンによる実装（Goで最も一般的）

```go
package main

import "fmt"

// User は構築されるオブジェクト
type User struct {
    name     string
    age      int
    email    string
    phone    string
    address  string
    isActive bool
}

func (u *User) String() string {
    return fmt.Sprintf("User{name=%s, age=%d, email=%s, phone=%s, address=%s, active=%v}",
        u.name, u.age, u.email, u.phone, u.address, u.isActive)
}

// UserBuilder はUserを構築するビルダー
type UserBuilder struct {
    user *User
}

func NewUserBuilder(name string) *UserBuilder {
    return &UserBuilder{
        user: &User{name: name},
    }
}

func (b *UserBuilder) Age(age int) *UserBuilder {
    b.user.age = age
    return b
}

func (b *UserBuilder) Email(email string) *UserBuilder {
    b.user.email = email
    return b
}

func (b *UserBuilder) Phone(phone string) *UserBuilder {
    b.user.phone = phone
    return b
}

func (b *UserBuilder) Address(address string) *UserBuilder {
    b.user.address = address
    return b
}

func (b *UserBuilder) IsActive(active bool) *UserBuilder {
    b.user.isActive = active
    return b
}

func (b *UserBuilder) Build() *User {
    return b.user
}

func main() {
    // メソッドチェーンで可読性の高い構築
    user := NewUserBuilder("田中太郎").
        Age(30).
        Email("tanaka@example.com").
        Phone("090-1234-5678").
        Address("東京都渋谷区").
        IsActive(true).
        Build()

    fmt.Println(user)

    // 一部のフィールドだけ設定することも可能
    simpleUser := NewUserBuilder("山田花子").
        Email("yamada@example.com").
        Build()

    fmt.Println(simpleUser)
}
```

### Functional Optionsパターン（Goでよく使われる）

```go
package main

import "fmt"

type Server struct {
    host    string
    port    int
    timeout int
    maxConn int
    tls     bool
}

// Option はServerの設定オプション
type Option func(*Server)

// With関数でオプションを定義
func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(timeout int) Option {
    return func(s *Server) {
        s.timeout = timeout
    }
}

func WithMaxConn(maxConn int) Option {
    return func(s *Server) {
        s.maxConn = maxConn
    }
}

func WithTLS(tls bool) Option {
    return func(s *Server) {
        s.tls = tls
    }
}

// NewServer はデフォルト値を持つServerを生成
func NewServer(host string, opts ...Option) *Server {
    // デフォルト値を設定
    server := &Server{
        host:    host,
        port:    8080,
        timeout: 30,
        maxConn: 100,
        tls:     false,
    }

    // オプションを適用
    for _, opt := range opts {
        opt(server)
    }

    return server
}

func main() {
    // デフォルト設定
    s1 := NewServer("localhost")
    fmt.Printf("Server 1: %+v\n", s1)

    // カスタム設定
    s2 := NewServer("api.example.com",
        WithPort(443),
        WithTLS(true),
        WithTimeout(60),
        WithMaxConn(1000),
    )
    fmt.Printf("Server 2: %+v\n", s2)
}
```

### Director を使った実装

```go
package main

import "fmt"

// House は構築される製品
type House struct {
    foundation string
    structure  string
    roof       string
    interior   string
}

func (h *House) String() string {
    return fmt.Sprintf("House{foundation=%s, structure=%s, roof=%s, interior=%s}",
        h.foundation, h.structure, h.roof, h.interior)
}

// HouseBuilder はHouseを構築するインターフェース
type HouseBuilder interface {
    BuildFoundation()
    BuildStructure()
    BuildRoof()
    BuildInterior()
    GetHouse() *House
}

// WoodenHouseBuilder は木造住宅のビルダー
type WoodenHouseBuilder struct {
    house *House
}

func NewWoodenHouseBuilder() *WoodenHouseBuilder {
    return &WoodenHouseBuilder{house: &House{}}
}

func (b *WoodenHouseBuilder) BuildFoundation() {
    b.house.foundation = "木製の基礎"
}

func (b *WoodenHouseBuilder) BuildStructure() {
    b.house.structure = "木造の骨組み"
}

func (b *WoodenHouseBuilder) BuildRoof() {
    b.house.roof = "木製の屋根"
}

func (b *WoodenHouseBuilder) BuildInterior() {
    b.house.interior = "木のインテリア"
}

func (b *WoodenHouseBuilder) GetHouse() *House {
    return b.house
}

// ConcreteHouseBuilder は鉄筋コンクリート住宅のビルダー
type ConcreteHouseBuilder struct {
    house *House
}

func NewConcreteHouseBuilder() *ConcreteHouseBuilder {
    return &ConcreteHouseBuilder{house: &House{}}
}

func (b *ConcreteHouseBuilder) BuildFoundation() {
    b.house.foundation = "コンクリート基礎"
}

func (b *ConcreteHouseBuilder) BuildStructure() {
    b.house.structure = "鉄筋コンクリート構造"
}

func (b *ConcreteHouseBuilder) BuildRoof() {
    b.house.roof = "コンクリート屋根"
}

func (b *ConcreteHouseBuilder) BuildInterior() {
    b.house.interior = "モダンインテリア"
}

func (b *ConcreteHouseBuilder) GetHouse() *House {
    return b.house
}

// Director は構築プロセスを管理
type Director struct {
    builder HouseBuilder
}

func NewDirector(builder HouseBuilder) *Director {
    return &Director{builder: builder}
}

func (d *Director) SetBuilder(builder HouseBuilder) {
    d.builder = builder
}

func (d *Director) ConstructHouse() *House {
    d.builder.BuildFoundation()
    d.builder.BuildStructure()
    d.builder.BuildRoof()
    d.builder.BuildInterior()
    return d.builder.GetHouse()
}

func main() {
    // 木造住宅を構築
    woodenBuilder := NewWoodenHouseBuilder()
    director := NewDirector(woodenBuilder)
    woodenHouse := director.ConstructHouse()
    fmt.Println("木造住宅:", woodenHouse)

    // 鉄筋コンクリート住宅を構築
    concreteBuilder := NewConcreteHouseBuilder()
    director.SetBuilder(concreteBuilder)
    concreteHouse := director.ConstructHouse()
    fmt.Println("RC住宅:", concreteHouse)
}
```

## 実践的な例：HTTPリクエストビルダー

```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "net/http"
    "net/url"
)

// Request は構築されるHTTPリクエスト
type Request struct {
    method  string
    url     string
    headers map[string]string
    query   url.Values
    body    io.Reader
}

// RequestBuilder はHTTPリクエストを構築する
type RequestBuilder struct {
    request *Request
    err     error
}

func NewRequestBuilder() *RequestBuilder {
    return &RequestBuilder{
        request: &Request{
            method:  "GET",
            headers: make(map[string]string),
            query:   make(url.Values),
        },
    }
}

func (b *RequestBuilder) Method(method string) *RequestBuilder {
    b.request.method = method
    return b
}

func (b *RequestBuilder) URL(rawURL string) *RequestBuilder {
    b.request.url = rawURL
    return b
}

func (b *RequestBuilder) Header(key, value string) *RequestBuilder {
    b.request.headers[key] = value
    return b
}

func (b *RequestBuilder) Query(key, value string) *RequestBuilder {
    b.request.query.Add(key, value)
    return b
}

func (b *RequestBuilder) Body(body []byte) *RequestBuilder {
    b.request.body = bytes.NewReader(body)
    return b
}

func (b *RequestBuilder) JSON(body []byte) *RequestBuilder {
    b.request.headers["Content-Type"] = "application/json"
    b.request.body = bytes.NewReader(body)
    return b
}

func (b *RequestBuilder) Bearer(token string) *RequestBuilder {
    b.request.headers["Authorization"] = "Bearer " + token
    return b
}

func (b *RequestBuilder) Build() (*http.Request, error) {
    // URLにクエリパラメータを追加
    finalURL := b.request.url
    if len(b.request.query) > 0 {
        finalURL += "?" + b.request.query.Encode()
    }

    req, err := http.NewRequest(b.request.method, finalURL, b.request.body)
    if err != nil {
        return nil, err
    }

    for key, value := range b.request.headers {
        req.Header.Set(key, value)
    }

    return req, nil
}

func main() {
    // GETリクエスト
    getReq, _ := NewRequestBuilder().
        URL("https://api.example.com/users").
        Query("page", "1").
        Query("limit", "10").
        Header("Accept", "application/json").
        Bearer("my-token").
        Build()

    fmt.Printf("GET: %s\n", getReq.URL)

    // POSTリクエスト
    postReq, _ := NewRequestBuilder().
        Method("POST").
        URL("https://api.example.com/users").
        JSON([]byte(`{"name": "田中太郎"}`)).
        Bearer("my-token").
        Build()

    fmt.Printf("POST: %s, Content-Type: %s\n",
        postReq.URL, postReq.Header.Get("Content-Type"))
}
```

## 実践的な例：SQLクエリビルダー

```go
package main

import (
    "fmt"
    "strings"
)

type QueryBuilder struct {
    table      string
    columns    []string
    conditions []string
    orderBy    string
    limit      int
    offset     int
}

func NewQueryBuilder() *QueryBuilder {
    return &QueryBuilder{
        columns: []string{"*"},
    }
}

func (b *QueryBuilder) Select(columns ...string) *QueryBuilder {
    b.columns = columns
    return b
}

func (b *QueryBuilder) From(table string) *QueryBuilder {
    b.table = table
    return b
}

func (b *QueryBuilder) Where(condition string) *QueryBuilder {
    b.conditions = append(b.conditions, condition)
    return b
}

func (b *QueryBuilder) OrderBy(column string) *QueryBuilder {
    b.orderBy = column
    return b
}

func (b *QueryBuilder) Limit(limit int) *QueryBuilder {
    b.limit = limit
    return b
}

func (b *QueryBuilder) Offset(offset int) *QueryBuilder {
    b.offset = offset
    return b
}

func (b *QueryBuilder) Build() string {
    var sb strings.Builder

    sb.WriteString("SELECT ")
    sb.WriteString(strings.Join(b.columns, ", "))
    sb.WriteString(" FROM ")
    sb.WriteString(b.table)

    if len(b.conditions) > 0 {
        sb.WriteString(" WHERE ")
        sb.WriteString(strings.Join(b.conditions, " AND "))
    }

    if b.orderBy != "" {
        sb.WriteString(" ORDER BY ")
        sb.WriteString(b.orderBy)
    }

    if b.limit > 0 {
        sb.WriteString(fmt.Sprintf(" LIMIT %d", b.limit))
    }

    if b.offset > 0 {
        sb.WriteString(fmt.Sprintf(" OFFSET %d", b.offset))
    }

    return sb.String()
}

func main() {
    query := NewQueryBuilder().
        Select("id", "name", "email").
        From("users").
        Where("age >= 18").
        Where("status = 'active'").
        OrderBy("created_at DESC").
        Limit(10).
        Offset(20).
        Build()

    fmt.Println(query)
    // SELECT id, name, email FROM users WHERE age >= 18 AND status = 'active' ORDER BY created_at DESC LIMIT 10 OFFSET 20
}
```

## Goでのポイント

### メソッドチェーンのパターン

```go
// 各メソッドでビルダー自身を返す
func (b *Builder) SetX(x int) *Builder {
    b.x = x
    return b
}
```

### エラーハンドリング

構築中のエラーをビルダーに保持し、Build時にまとめて返す：

```go
type Builder struct {
    result *Result
    err    error
}

func (b *Builder) Step1() *Builder {
    if b.err != nil {
        return b // 既にエラーがある場合はスキップ
    }
    // 処理...
    return b
}

func (b *Builder) Build() (*Result, error) {
    if b.err != nil {
        return nil, b.err
    }
    return b.result, nil
}
```

### 不変オブジェクトの生成

```go
type ImmutableConfig struct {
    host string
    port int
}

func (b *ConfigBuilder) Build() ImmutableConfig {
    // ポインタではなく値を返すことで不変性を保証
    return ImmutableConfig{
        host: b.host,
        port: b.port,
    }
}
```

## まとめ

- Builderパターンは、複雑なオブジェクトを段階的に構築する
- Goでは**メソッドチェーン**や**Functional Options**が一般的
- 可読性が高く、オプションパラメータの管理が容易になる

次章では、既存オブジェクトをコピーして新規作成するPrototypeパターンを学びます。
