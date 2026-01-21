---
title: "Facade パターン"
---

# Facade パターン

## 概要

Facadeパターンは、複雑なサブシステムへのシンプルなインターフェースを提供するパターンです。サブシステムの複雑さを隠蔽し、クライアントが簡単に利用できるようにします。

## 問題

以下のような状況で使用を検討します：

- 複雑なライブラリやフレームワークの使い方を簡略化したい
- サブシステムの依存関係をクライアントから隠したい
- レイヤー化されたアーキテクチャを構築したい

## 構造

```
┌─────────────┐
│   Client    │
└─────────────┘
       │
       ▼
┌─────────────┐
│   Facade    │
├─────────────┤      ┌───────────────────────────┐
│ + Operation │─────▶│      Subsystem            │
└─────────────┘      │ ┌─────┐ ┌─────┐ ┌─────┐  │
                     │ │Class│ │Class│ │Class│  │
                     │ │  A  │ │  B  │ │  C  │  │
                     │ └─────┘ └─────┘ └─────┘  │
                     └───────────────────────────┘
```

## 実装

### ホームシアターシステムの例

```go
package main

import "fmt"

// ===== サブシステムのクラス群 =====

// Amplifier はアンプ
type Amplifier struct {
    volume int
}

func (a *Amplifier) On() {
    fmt.Println("アンプの電源をONにしました")
}

func (a *Amplifier) Off() {
    fmt.Println("アンプの電源をOFFにしました")
}

func (a *Amplifier) SetVolume(level int) {
    a.volume = level
    fmt.Printf("アンプの音量を%dに設定しました\n", level)
}

// DVDPlayer はDVDプレイヤー
type DVDPlayer struct{}

func (d *DVDPlayer) On() {
    fmt.Println("DVDプレイヤーの電源をONにしました")
}

func (d *DVDPlayer) Off() {
    fmt.Println("DVDプレイヤーの電源をOFFにしました")
}

func (d *DVDPlayer) Play(movie string) {
    fmt.Printf("「%s」を再生します\n", movie)
}

func (d *DVDPlayer) Stop() {
    fmt.Println("DVDプレイヤーを停止しました")
}

// Projector はプロジェクター
type Projector struct{}

func (p *Projector) On() {
    fmt.Println("プロジェクターの電源をONにしました")
}

func (p *Projector) Off() {
    fmt.Println("プロジェクターの電源をOFFにしました")
}

func (p *Projector) SetWideScreenMode() {
    fmt.Println("プロジェクターをワイドスクリーンモードに設定しました")
}

// Lights は照明
type Lights struct{}

func (l *Lights) Dim(level int) {
    fmt.Printf("照明を%d%%に調光しました\n", level)
}

func (l *Lights) On() {
    fmt.Println("照明をONにしました")
}

// ===== Facade =====

// HomeTheaterFacade はホームシアターのファサード
type HomeTheaterFacade struct {
    amp       *Amplifier
    dvd       *DVDPlayer
    projector *Projector
    lights    *Lights
}

func NewHomeTheaterFacade() *HomeTheaterFacade {
    return &HomeTheaterFacade{
        amp:       &Amplifier{},
        dvd:       &DVDPlayer{},
        projector: &Projector{},
        lights:    &Lights{},
    }
}

// WatchMovie は映画を見るための一連の操作
func (h *HomeTheaterFacade) WatchMovie(movie string) {
    fmt.Println("映画を見る準備をします...")
    h.lights.Dim(10)
    h.projector.On()
    h.projector.SetWideScreenMode()
    h.amp.On()
    h.amp.SetVolume(5)
    h.dvd.On()
    h.dvd.Play(movie)
    fmt.Println("映画をお楽しみください！")
}

// EndMovie は映画を終了するための一連の操作
func (h *HomeTheaterFacade) EndMovie() {
    fmt.Println("\n映画を終了します...")
    h.dvd.Stop()
    h.dvd.Off()
    h.amp.Off()
    h.projector.Off()
    h.lights.On()
    fmt.Println("終了しました")
}

func main() {
    // クライアントはFacadeだけを使う
    homeTheater := NewHomeTheaterFacade()
    homeTheater.WatchMovie("となりのトトロ")
    homeTheater.EndMovie()
}
```

### コンピュータ起動の例

```go
package main

import "fmt"

// ===== サブシステム =====

type CPU struct{}

func (c *CPU) Freeze()         { fmt.Println("CPU: フリーズ") }
func (c *CPU) Jump(addr int64) { fmt.Printf("CPU: アドレス %x にジャンプ\n", addr) }
func (c *CPU) Execute()        { fmt.Println("CPU: 実行開始") }

type Memory struct{}

func (m *Memory) Load(addr int64, data []byte) {
    fmt.Printf("Memory: アドレス %x に %d bytes をロード\n", addr, len(data))
}

type HardDrive struct{}

func (h *HardDrive) Read(sector, size int64) []byte {
    fmt.Printf("HardDrive: セクタ %d から %d bytes を読み込み\n", sector, size)
    return make([]byte, size)
}

// ===== Facade =====

const BootAddress int64 = 0x7C00
const BootSector int64 = 0
const SectorSize int64 = 512

type ComputerFacade struct {
    cpu       *CPU
    memory    *Memory
    hardDrive *HardDrive
}

func NewComputer() *ComputerFacade {
    return &ComputerFacade{
        cpu:       &CPU{},
        memory:    &Memory{},
        hardDrive: &HardDrive{},
    }
}

func (c *ComputerFacade) Start() {
    fmt.Println("=== コンピュータを起動します ===")
    c.cpu.Freeze()
    bootData := c.hardDrive.Read(BootSector, SectorSize)
    c.memory.Load(BootAddress, bootData)
    c.cpu.Jump(BootAddress)
    c.cpu.Execute()
    fmt.Println("=== 起動完了 ===")
}

func main() {
    computer := NewComputer()
    computer.Start()
}
```

## 実践的な例：注文処理システム

```go
package main

import "fmt"

// ===== サブシステム =====

// InventoryService は在庫サービス
type InventoryService struct{}

func (i *InventoryService) CheckStock(productID string) bool {
    fmt.Printf("[Inventory] 商品 %s の在庫を確認中...\n", productID)
    return true
}

func (i *InventoryService) ReserveStock(productID string, qty int) error {
    fmt.Printf("[Inventory] 商品 %s を %d 個予約しました\n", productID, qty)
    return nil
}

// PaymentService は決済サービス
type PaymentService struct{}

func (p *PaymentService) ProcessPayment(amount float64, cardNumber string) error {
    fmt.Printf("[Payment] ¥%.0f の決済を処理中...\n", amount)
    fmt.Println("[Payment] 決済完了")
    return nil
}

// ShippingService は配送サービス
type ShippingService struct{}

func (s *ShippingService) CalculateShippingCost(address string) float64 {
    fmt.Printf("[Shipping] %s への配送料を計算中...\n", address)
    return 500
}

func (s *ShippingService) ScheduleDelivery(orderID, address string) error {
    fmt.Printf("[Shipping] 注文 %s の配送を %s に手配しました\n", orderID, address)
    return nil
}

// NotificationService は通知サービス
type NotificationService struct{}

func (n *NotificationService) SendOrderConfirmation(email, orderID string) {
    fmt.Printf("[Notification] %s に注文確認メールを送信しました (注文: %s)\n", email, orderID)
}

func (n *NotificationService) SendShippingNotification(email, trackingNumber string) {
    fmt.Printf("[Notification] %s に発送通知を送信しました (追跡番号: %s)\n", email, trackingNumber)
}

// ===== Facade =====

// Order は注文情報
type Order struct {
    ID        string
    ProductID string
    Quantity  int
    Amount    float64
    Email     string
    Address   string
    Card      string
}

// OrderFacade は注文処理のファサード
type OrderFacade struct {
    inventory    *InventoryService
    payment      *PaymentService
    shipping     *ShippingService
    notification *NotificationService
}

func NewOrderFacade() *OrderFacade {
    return &OrderFacade{
        inventory:    &InventoryService{},
        payment:      &PaymentService{},
        shipping:     &ShippingService{},
        notification: &NotificationService{},
    }
}

// PlaceOrder は注文を処理する
func (f *OrderFacade) PlaceOrder(order Order) error {
    fmt.Println("=== 注文処理を開始します ===\n")

    // 1. 在庫確認
    if !f.inventory.CheckStock(order.ProductID) {
        return fmt.Errorf("在庫がありません")
    }

    // 2. 配送料計算
    shippingCost := f.shipping.CalculateShippingCost(order.Address)
    totalAmount := order.Amount + shippingCost
    fmt.Printf("\n商品: ¥%.0f + 送料: ¥%.0f = 合計: ¥%.0f\n\n", order.Amount, shippingCost, totalAmount)

    // 3. 決済処理
    if err := f.payment.ProcessPayment(totalAmount, order.Card); err != nil {
        return fmt.Errorf("決済に失敗しました: %w", err)
    }

    // 4. 在庫予約
    if err := f.inventory.ReserveStock(order.ProductID, order.Quantity); err != nil {
        return fmt.Errorf("在庫予約に失敗しました: %w", err)
    }

    // 5. 配送手配
    if err := f.shipping.ScheduleDelivery(order.ID, order.Address); err != nil {
        return fmt.Errorf("配送手配に失敗しました: %w", err)
    }

    // 6. 確認メール送信
    f.notification.SendOrderConfirmation(order.Email, order.ID)

    fmt.Println("\n=== 注文処理が完了しました ===")
    return nil
}

func main() {
    facade := NewOrderFacade()

    order := Order{
        ID:        "ORD-001",
        ProductID: "PROD-123",
        Quantity:  2,
        Amount:    5000,
        Email:     "customer@example.com",
        Address:   "東京都渋谷区...",
        Card:      "4111-1111-1111-1111",
    }

    if err := facade.PlaceOrder(order); err != nil {
        fmt.Printf("エラー: %v\n", err)
    }
}
```

## 実践的な例：データベース操作

```go
package main

import (
    "fmt"
)

// ===== サブシステム（簡略化）=====

type ConnectionPool struct{}

func (c *ConnectionPool) GetConnection() string {
    return "conn-1"
}

func (c *ConnectionPool) ReleaseConnection(conn string) {}

type QueryBuilder struct{}

func (q *QueryBuilder) Select(table string, fields []string) string {
    return fmt.Sprintf("SELECT %v FROM %s", fields, table)
}

func (q *QueryBuilder) Insert(table string, data map[string]any) string {
    return fmt.Sprintf("INSERT INTO %s VALUES %v", table, data)
}

type ResultMapper struct{}

func (r *ResultMapper) MapToStruct(data map[string]any, target any) error {
    return nil
}

// ===== Facade =====

type DatabaseFacade struct {
    pool    *ConnectionPool
    builder *QueryBuilder
    mapper  *ResultMapper
}

func NewDatabaseFacade() *DatabaseFacade {
    return &DatabaseFacade{
        pool:    &ConnectionPool{},
        builder: &QueryBuilder{},
        mapper:  &ResultMapper{},
    }
}

func (d *DatabaseFacade) FindByID(table string, id int) (map[string]any, error) {
    conn := d.pool.GetConnection()
    defer d.pool.ReleaseConnection(conn)

    query := d.builder.Select(table, []string{"*"})
    query += fmt.Sprintf(" WHERE id = %d", id)

    fmt.Printf("Executing: %s\n", query)

    // 実際にはここでクエリを実行
    return map[string]any{"id": id, "name": "sample"}, nil
}

func (d *DatabaseFacade) Save(table string, data map[string]any) error {
    conn := d.pool.GetConnection()
    defer d.pool.ReleaseConnection(conn)

    query := d.builder.Insert(table, data)
    fmt.Printf("Executing: %s\n", query)

    return nil
}

func main() {
    db := NewDatabaseFacade()

    // シンプルなAPI
    user, _ := db.FindByID("users", 1)
    fmt.Printf("Found: %v\n", user)

    db.Save("users", map[string]any{"name": "新規ユーザー", "email": "new@example.com"})
}
```

## Goでのポイント

### パッケージレベルのFacade

Goではパッケージ自体がFacadeの役割を果たすことがあります：

```go
// mypackage/facade.go
package mypackage

// 公開API（Facade）
func DoSomething() error {
    // 内部の複雑な処理を隠蔽
    return internalProcess()
}

// 非公開の内部処理
func internalProcess() error {
    // 複雑なサブシステムの処理
    return nil
}
```

### 設定の簡略化

```go
// 複雑な設定をFacadeで簡略化
func NewDefaultServer() *Server {
    return &Server{
        Config:     DefaultConfig(),
        Logger:     NewDefaultLogger(),
        Router:     NewDefaultRouter(),
        Middleware: DefaultMiddleware(),
    }
}
```

## まとめ

- Facadeは複雑なサブシステムへのシンプルなインターフェースを提供
- クライアントとサブシステムの結合度を下げる
- Goではパッケージ設計でも活用できる

次章では、多数のオブジェクトを効率的に共有するFlyweightパターンを学びます。
