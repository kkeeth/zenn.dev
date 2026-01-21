---
title: "Factory Method パターン"
---

# Factory Method パターン

## 概要

Factory Methodパターンは、オブジェクトの生成をサブクラス（または別の関数）に委譲するパターンです。生成するオブジェクトの具体的な型を、使用するコードから分離します。

## 問題

以下のような状況で使用を検討します：

- 生成するオブジェクトの型を実行時に決定したい
- オブジェクトの生成ロジックを一箇所にまとめたい
- 新しい種類のオブジェクトを追加しやすくしたい

## 構造

```
┌─────────────────┐         ┌─────────────────┐
│    Product      │◁────────│ ConcreteProduct │
│  (interface)    │         │   (struct)      │
├─────────────────┤         ├─────────────────┤
│ + Operation()   │         │ + Operation()   │
└─────────────────┘         └─────────────────┘
        △
        │
┌─────────────────┐         ┌─────────────────┐
│    Creator      │◁────────│ ConcreteCreator │
│  (interface)    │         │   (struct)      │
├─────────────────┤         ├─────────────────┤
│ + CreateProduct │         │ + CreateProduct │
└─────────────────┘         └─────────────────┘
```

## 実装

### 基本的な実装

```go
package main

import "fmt"

// Document はドキュメントを表すインターフェース
type Document interface {
    Open()
    Save()
    Close()
}

// PDFDocument はPDFドキュメント
type PDFDocument struct {
    filename string
}

func (d *PDFDocument) Open() {
    fmt.Printf("PDFドキュメント '%s' を開きました\n", d.filename)
}

func (d *PDFDocument) Save() {
    fmt.Printf("PDFドキュメント '%s' を保存しました\n", d.filename)
}

func (d *PDFDocument) Close() {
    fmt.Printf("PDFドキュメント '%s' を閉じました\n", d.filename)
}

// WordDocument はWordドキュメント
type WordDocument struct {
    filename string
}

func (d *WordDocument) Open() {
    fmt.Printf("Wordドキュメント '%s' を開きました\n", d.filename)
}

func (d *WordDocument) Save() {
    fmt.Printf("Wordドキュメント '%s' を保存しました\n", d.filename)
}

func (d *WordDocument) Close() {
    fmt.Printf("Wordドキュメント '%s' を閉じました\n", d.filename)
}

// Factory Method: ファイル名から適切なドキュメントを生成
func CreateDocument(filename string) Document {
    // 拡張子で判断
    if len(filename) > 4 && filename[len(filename)-4:] == ".pdf" {
        return &PDFDocument{filename: filename}
    }
    return &WordDocument{filename: filename}
}

func main() {
    // クライアントコードは具体的な型を知らなくてよい
    doc1 := CreateDocument("report.pdf")
    doc1.Open()
    doc1.Save()
    doc1.Close()

    doc2 := CreateDocument("letter.docx")
    doc2.Open()
    doc2.Save()
    doc2.Close()
}
```

### インターフェースを使った実装

より柔軟な実装として、Factory自体もインターフェースにする方法があります：

```go
package main

import "fmt"

// Transport は輸送手段を表すインターフェース
type Transport interface {
    Deliver()
}

// Truck はトラック輸送
type Truck struct{}

func (t *Truck) Deliver() {
    fmt.Println("トラックで陸路を輸送します")
}

// Ship は船舶輸送
type Ship struct{}

func (s *Ship) Deliver() {
    fmt.Println("船で海路を輸送します")
}

// Logistics は物流を表すインターフェース（Factory）
type Logistics interface {
    CreateTransport() Transport
    PlanDelivery()
}

// RoadLogistics は陸路物流
type RoadLogistics struct{}

func (r *RoadLogistics) CreateTransport() Transport {
    return &Truck{}
}

func (r *RoadLogistics) PlanDelivery() {
    transport := r.CreateTransport()
    fmt.Println("陸路の配送を計画中...")
    transport.Deliver()
}

// SeaLogistics は海路物流
type SeaLogistics struct{}

func (s *SeaLogistics) CreateTransport() Transport {
    return &Ship{}
}

func (s *SeaLogistics) PlanDelivery() {
    transport := s.CreateTransport()
    fmt.Println("海路の配送を計画中...")
    transport.Deliver()
}

func main() {
    // 使用する物流方法を選択
    var logistics Logistics

    logistics = &RoadLogistics{}
    logistics.PlanDelivery()

    logistics = &SeaLogistics{}
    logistics.PlanDelivery()
}
```

### 関数型での実装

Goでは、関数を使ったシンプルな実装も一般的です：

```go
package main

import "fmt"

// DB はデータベース接続を表すインターフェース
type DB interface {
    Connect() error
    Query(sql string) ([]map[string]any, error)
    Close() error
}

// MySQL 実装
type MySQL struct {
    host string
}

func (m *MySQL) Connect() error {
    fmt.Printf("MySQLに接続: %s\n", m.host)
    return nil
}

func (m *MySQL) Query(sql string) ([]map[string]any, error) {
    fmt.Printf("MySQL Query: %s\n", sql)
    return nil, nil
}

func (m *MySQL) Close() error {
    fmt.Println("MySQL接続を閉じました")
    return nil
}

// PostgreSQL 実装
type PostgreSQL struct {
    host string
}

func (p *PostgreSQL) Connect() error {
    fmt.Printf("PostgreSQLに接続: %s\n", p.host)
    return nil
}

func (p *PostgreSQL) Query(sql string) ([]map[string]any, error) {
    fmt.Printf("PostgreSQL Query: %s\n", sql)
    return nil, nil
}

func (p *PostgreSQL) Close() error {
    fmt.Println("PostgreSQL接続を閉じました")
    return nil
}

// Factory関数の型
type DBFactory func(host string) DB

// 各データベース用のFactory関数
func NewMySQL(host string) DB {
    return &MySQL{host: host}
}

func NewPostgreSQL(host string) DB {
    return &PostgreSQL{host: host}
}

// Factory関数を返す関数
func GetDBFactory(dbType string) DBFactory {
    switch dbType {
    case "mysql":
        return NewMySQL
    case "postgresql":
        return NewPostgreSQL
    default:
        return NewMySQL
    }
}

func main() {
    // 設定に応じてFactory関数を取得
    dbType := "postgresql"
    factory := GetDBFactory(dbType)

    // Factoryを使ってDBを生成
    db := factory("localhost:5432")
    db.Connect()
    db.Query("SELECT * FROM users")
    db.Close()
}
```

## 実践的な例：通知システム

```go
package main

import "fmt"

// Notifier は通知を送信するインターフェース
type Notifier interface {
    Send(message string) error
    GetType() string
}

// EmailNotifier はメール通知
type EmailNotifier struct {
    email string
}

func (n *EmailNotifier) Send(message string) error {
    fmt.Printf("[Email to %s]: %s\n", n.email, message)
    return nil
}

func (n *EmailNotifier) GetType() string {
    return "email"
}

// SMSNotifier はSMS通知
type SMSNotifier struct {
    phone string
}

func (n *SMSNotifier) Send(message string) error {
    fmt.Printf("[SMS to %s]: %s\n", n.phone, message)
    return nil
}

func (n *SMSNotifier) GetType() string {
    return "sms"
}

// SlackNotifier はSlack通知
type SlackNotifier struct {
    channel string
}

func (n *SlackNotifier) Send(message string) error {
    fmt.Printf("[Slack #%s]: %s\n", n.channel, message)
    return nil
}

func (n *SlackNotifier) GetType() string {
    return "slack"
}

// NotifierFactory は通知を生成するファクトリ
type NotifierFactory struct{}

// Create は指定されたタイプの通知を生成する
func (f *NotifierFactory) Create(notifierType string, target string) (Notifier, error) {
    switch notifierType {
    case "email":
        return &EmailNotifier{email: target}, nil
    case "sms":
        return &SMSNotifier{phone: target}, nil
    case "slack":
        return &SlackNotifier{channel: target}, nil
    default:
        return nil, fmt.Errorf("未対応の通知タイプ: %s", notifierType)
    }
}

// NotificationService は通知サービス
type NotificationService struct {
    factory *NotifierFactory
}

func NewNotificationService() *NotificationService {
    return &NotificationService{
        factory: &NotifierFactory{},
    }
}

func (s *NotificationService) Notify(notifierType, target, message string) error {
    notifier, err := s.factory.Create(notifierType, target)
    if err != nil {
        return err
    }
    return notifier.Send(message)
}

func main() {
    service := NewNotificationService()

    // 様々な方法で通知を送信
    service.Notify("email", "user@example.com", "メールでの通知です")
    service.Notify("sms", "090-1234-5678", "SMSでの通知です")
    service.Notify("slack", "general", "Slackでの通知です")
}
```

## Goでのポイント

### インターフェースを活用する

GoではFactory Methodパターンを実装する際、インターフェースが重要な役割を果たします。

```go
// 良い例：インターフェースを返す
func CreatePayment(method string) PaymentProcessor {
    // ...
}

// 悪い例：具体的な型を返す
func CreateCreditCardPayment() *CreditCardPayment {
    // ...
}
```

### 登録パターン

新しい型を動的に追加できるようにする場合、登録パターンを使います：

```go
package main

import "fmt"

type Creator func() Product

var registry = make(map[string]Creator)

func Register(name string, creator Creator) {
    registry[name] = creator
}

func Create(name string) Product {
    if creator, ok := registry[name]; ok {
        return creator()
    }
    return nil
}

type Product interface {
    Use()
}

type ProductA struct{}
func (p *ProductA) Use() { fmt.Println("ProductA") }

type ProductB struct{}
func (p *ProductB) Use() { fmt.Println("ProductB") }

func init() {
    Register("A", func() Product { return &ProductA{} })
    Register("B", func() Product { return &ProductB{} })
}

func main() {
    product := Create("A")
    product.Use()
}
```

## Simple Factory との違い

**Simple Factory**（静的ファクトリメソッド）は、単一のファクトリ関数でオブジェクトを生成します。**Factory Method**は、生成ロジック自体を拡張可能にするパターンです。

```go
// Simple Factory
func CreateAnimal(animalType string) Animal {
    // 1つの関数で全てを処理
}

// Factory Method
type AnimalFactory interface {
    CreateAnimal() Animal
}

type DogFactory struct{}
func (f *DogFactory) CreateAnimal() Animal { return &Dog{} }

type CatFactory struct{}
func (f *CatFactory) CreateAnimal() Animal { return &Cat{} }
```

## まとめ

- Factory Methodは、オブジェクトの生成をカプセル化するパターン
- Goではインターフェースと関数を使ってシンプルに実装できる
- 新しい型を追加しやすく、クライアントコードの変更が不要になる

次章では、関連するオブジェクト群をまとめて生成するAbstract Factoryパターンを学びます。
