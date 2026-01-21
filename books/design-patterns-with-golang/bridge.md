---
title: "Bridge パターン"
---

# Bridge パターン

## 概要

Bridgeパターンは、抽象化（Abstraction）と実装（Implementation）を分離し、それぞれを独立して変更できるようにするパターンです。

## 問題

以下のような状況で使用を検討します：

- 抽象と実装の両方が拡張される可能性がある
- 実装の詳細をクライアントから隠蔽したい
- 実行時に実装を切り替えたい

## Adapterとの違い

| 観点 | Adapter | Bridge |
|-----|---------|--------|
| タイミング | 後から適用（既存コードの統合） | 設計時に適用 |
| 目的 | 互換性のないインターフェースを接続 | 抽象と実装を分離 |
| 変更 | 一方を他方に合わせる | 両者を独立して拡張 |

## 構造

```
┌─────────────────┐           ┌─────────────────┐
│   Abstraction   │──────────▶│ Implementor     │
├─────────────────┤           │   (interface)   │
│ + Operation()   │           ├─────────────────┤
└─────────────────┘           │ + OperationImpl │
        △                     └─────────────────┘
        │                             △
        │                       ┌─────┴─────┐
┌───────────────┐         ┌─────┴───┐   ┌───┴─────┐
│RefinedAbstract│         │ ImplA   │   │ ImplB   │
└───────────────┘         └─────────┘   └─────────┘
```

## 実装

### 基本的な実装

```go
package main

import "fmt"

// ===== Implementor（実装側のインターフェース）=====

// Device はデバイスの基本操作を定義
type Device interface {
    IsEnabled() bool
    Enable()
    Disable()
    GetVolume() int
    SetVolume(volume int)
}

// ===== Concrete Implementors =====

// TV はテレビの実装
type TV struct {
    enabled bool
    volume  int
}

func (t *TV) IsEnabled() bool { return t.enabled }
func (t *TV) Enable()         { t.enabled = true; fmt.Println("TV: 電源ON") }
func (t *TV) Disable()        { t.enabled = false; fmt.Println("TV: 電源OFF") }
func (t *TV) GetVolume() int  { return t.volume }
func (t *TV) SetVolume(v int) { t.volume = v; fmt.Printf("TV: 音量 %d\n", v) }

// Radio はラジオの実装
type Radio struct {
    enabled bool
    volume  int
}

func (r *Radio) IsEnabled() bool { return r.enabled }
func (r *Radio) Enable()         { r.enabled = true; fmt.Println("Radio: 電源ON") }
func (r *Radio) Disable()        { r.enabled = false; fmt.Println("Radio: 電源OFF") }
func (r *Radio) GetVolume() int  { return r.volume }
func (r *Radio) SetVolume(v int) { r.volume = v; fmt.Printf("Radio: 音量 %d\n", v) }

// ===== Abstraction（抽象側）=====

// Remote はリモコンの抽象
type Remote struct {
    device Device
}

func NewRemote(device Device) *Remote {
    return &Remote{device: device}
}

func (r *Remote) TogglePower() {
    if r.device.IsEnabled() {
        r.device.Disable()
    } else {
        r.device.Enable()
    }
}

func (r *Remote) VolumeUp() {
    r.device.SetVolume(r.device.GetVolume() + 10)
}

func (r *Remote) VolumeDown() {
    r.device.SetVolume(r.device.GetVolume() - 10)
}

// ===== Refined Abstraction =====

// AdvancedRemote は高度な機能を持つリモコン
type AdvancedRemote struct {
    *Remote // 埋め込みで基本機能を継承
}

func NewAdvancedRemote(device Device) *AdvancedRemote {
    return &AdvancedRemote{
        Remote: NewRemote(device),
    }
}

func (r *AdvancedRemote) Mute() {
    r.device.SetVolume(0)
    fmt.Println("ミュートしました")
}

func main() {
    // TV + 通常リモコン
    fmt.Println("=== TV with Basic Remote ===")
    tv := &TV{}
    tvRemote := NewRemote(tv)
    tvRemote.TogglePower()
    tvRemote.VolumeUp()
    tvRemote.VolumeUp()

    // Radio + 高度なリモコン
    fmt.Println("\n=== Radio with Advanced Remote ===")
    radio := &Radio{}
    radioRemote := NewAdvancedRemote(radio)
    radioRemote.TogglePower()
    radioRemote.VolumeUp()
    radioRemote.Mute()
}
```

### 描画システムの例

```go
package main

import "fmt"

// ===== Implementor =====

// DrawingAPI は描画APIのインターフェース
type DrawingAPI interface {
    DrawCircle(x, y, radius float64)
    DrawRectangle(x, y, width, height float64)
}

// ===== Concrete Implementors =====

// SVGDrawing はSVG形式で描画
type SVGDrawing struct{}

func (s *SVGDrawing) DrawCircle(x, y, radius float64) {
    fmt.Printf("<circle cx=\"%.1f\" cy=\"%.1f\" r=\"%.1f\" />\n", x, y, radius)
}

func (s *SVGDrawing) DrawRectangle(x, y, width, height float64) {
    fmt.Printf("<rect x=\"%.1f\" y=\"%.1f\" width=\"%.1f\" height=\"%.1f\" />\n",
        x, y, width, height)
}

// CanvasDrawing はCanvas APIで描画
type CanvasDrawing struct{}

func (c *CanvasDrawing) DrawCircle(x, y, radius float64) {
    fmt.Printf("ctx.beginPath(); ctx.arc(%.1f, %.1f, %.1f, 0, 2*Math.PI); ctx.stroke();\n",
        x, y, radius)
}

func (c *CanvasDrawing) DrawRectangle(x, y, width, height float64) {
    fmt.Printf("ctx.strokeRect(%.1f, %.1f, %.1f, %.1f);\n", x, y, width, height)
}

// ===== Abstraction =====

// Shape は図形の抽象
type Shape interface {
    Draw()
    Resize(factor float64)
}

// ===== Refined Abstractions =====

// Circle は円
type Circle struct {
    x, y, radius float64
    api          DrawingAPI
}

func NewCircle(x, y, radius float64, api DrawingAPI) *Circle {
    return &Circle{x: x, y: y, radius: radius, api: api}
}

func (c *Circle) Draw() {
    c.api.DrawCircle(c.x, c.y, c.radius)
}

func (c *Circle) Resize(factor float64) {
    c.radius *= factor
}

// Rectangle は長方形
type Rectangle struct {
    x, y, width, height float64
    api                 DrawingAPI
}

func NewRectangle(x, y, width, height float64, api DrawingAPI) *Rectangle {
    return &Rectangle{x: x, y: y, width: width, height: height, api: api}
}

func (r *Rectangle) Draw() {
    r.api.DrawRectangle(r.x, r.y, r.width, r.height)
}

func (r *Rectangle) Resize(factor float64) {
    r.width *= factor
    r.height *= factor
}

func main() {
    fmt.Println("=== SVG出力 ===")
    svgAPI := &SVGDrawing{}
    circle1 := NewCircle(100, 100, 50, svgAPI)
    rect1 := NewRectangle(10, 10, 200, 100, svgAPI)
    circle1.Draw()
    rect1.Draw()

    fmt.Println("\n=== Canvas出力 ===")
    canvasAPI := &CanvasDrawing{}
    circle2 := NewCircle(100, 100, 50, canvasAPI)
    rect2 := NewRectangle(10, 10, 200, 100, canvasAPI)
    circle2.Draw()
    rect2.Draw()
}
```

## 実践的な例：通知システム

```go
package main

import "fmt"

// ===== Implementor =====

// MessageSender はメッセージ送信のインターフェース
type MessageSender interface {
    Send(to, message string) error
}

// ===== Concrete Implementors =====

type EmailSender struct{}

func (e *EmailSender) Send(to, message string) error {
    fmt.Printf("[Email] To: %s\nMessage: %s\n", to, message)
    return nil
}

type SMSSender struct{}

func (s *SMSSender) Send(to, message string) error {
    fmt.Printf("[SMS] To: %s\nMessage: %s\n", to, message)
    return nil
}

type SlackSender struct {
    webhookURL string
}

func (s *SlackSender) Send(to, message string) error {
    fmt.Printf("[Slack #%s] Message: %s\n", to, message)
    return nil
}

// ===== Abstraction =====

// Notification は通知の抽象
type Notification struct {
    sender MessageSender
}

func (n *Notification) Notify(recipient, message string) error {
    return n.sender.Send(recipient, message)
}

// ===== Refined Abstractions =====

// UrgentNotification は緊急通知
type UrgentNotification struct {
    *Notification
}

func NewUrgentNotification(sender MessageSender) *UrgentNotification {
    return &UrgentNotification{
        Notification: &Notification{sender: sender},
    }
}

func (n *UrgentNotification) Notify(recipient, message string) error {
    urgentMessage := fmt.Sprintf("🚨 緊急 🚨\n%s", message)
    return n.sender.Send(recipient, urgentMessage)
}

// ScheduledNotification は予約通知
type ScheduledNotification struct {
    *Notification
    scheduleTime string
}

func NewScheduledNotification(sender MessageSender, time string) *ScheduledNotification {
    return &ScheduledNotification{
        Notification: &Notification{sender: sender},
        scheduleTime: time,
    }
}

func (n *ScheduledNotification) Notify(recipient, message string) error {
    scheduledMessage := fmt.Sprintf("[予定: %s]\n%s", n.scheduleTime, message)
    return n.sender.Send(recipient, scheduledMessage)
}

func main() {
    fmt.Println("=== 通常メール通知 ===")
    emailNotif := &Notification{sender: &EmailSender{}}
    emailNotif.Notify("user@example.com", "会議のリマインダーです")

    fmt.Println("\n=== 緊急SMS通知 ===")
    urgentSMS := NewUrgentNotification(&SMSSender{})
    urgentSMS.Notify("090-1234-5678", "サーバーがダウンしています")

    fmt.Println("\n=== 予約Slack通知 ===")
    scheduledSlack := NewScheduledNotification(&SlackSender{}, "2024-01-15 09:00")
    scheduledSlack.Notify("general", "定例ミーティングのお知らせ")
}
```

## Goでのポイント

### インターフェースの小分け

Goではインターフェースを小さく保つことが推奨されます：

```go
// 大きなインターフェースを避ける
type BigInterface interface {
    Method1()
    Method2()
    Method3()
}

// 小さなインターフェースに分割
type Enabler interface {
    Enable()
    Disable()
}

type VolumeController interface {
    GetVolume() int
    SetVolume(int)
}

// 必要に応じて組み合わせ
type Device interface {
    Enabler
    VolumeController
}
```

### 依存性注入との組み合わせ

Bridgeパターンは依存性注入と相性が良いです：

```go
func NewService(db Database, cache Cache, logger Logger) *Service {
    return &Service{
        db:     db,
        cache:  cache,
        logger: logger,
    }
}
```

## まとめ

- Bridgeは抽象と実装を分離し、独立して拡張可能にする
- 「継承」ではなく「委譲」を使う
- Goのインターフェースと構造体で自然に実装できる

次章では、部分-全体の階層構造を表現するCompositeパターンを学びます。
