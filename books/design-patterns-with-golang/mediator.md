---
title: "Mediator パターン"
---

# Mediator パターン

## 概要

Mediatorパターンは、オブジェクト間の複雑な通信を、仲介者オブジェクトを通して整理するパターンです。オブジェクト同士が直接参照し合うことを避け、疎結合を実現します。

## 問題

以下のような状況で使用を検討します：

- 多数のオブジェクトが複雑に通信している
- オブジェクト間の依存関係を減らしたい
- 通信ロジックを一箇所にまとめたい

## 構造

```
┌───────────┐     ┌───────────┐
│Colleague A│◀───▶│  Mediator │◀───▶┌───────────┐
└───────────┘     └───────────┘     │Colleague B│
                       ▲            └───────────┘
                       │
                  ┌───────────┐
                  │Colleague C│
                  └───────────┘
```

## 実装

### チャットルームの例

```go
package main

import "fmt"

// ChatMediator はチャットの仲介者インターフェース
type ChatMediator interface {
    SendMessage(message string, sender *User)
    AddUser(user *User)
}

// ChatRoom はチャットルーム（具体的なMediator）
type ChatRoom struct {
    users []*User
}

func NewChatRoom() *ChatRoom {
    return &ChatRoom{users: []*User{}}
}

func (c *ChatRoom) AddUser(user *User) {
    c.users = append(c.users, user)
    user.chatRoom = c
}

func (c *ChatRoom) SendMessage(message string, sender *User) {
    for _, user := range c.users {
        if user != sender {
            user.Receive(message, sender.name)
        }
    }
}

// User はユーザー（Colleague）
type User struct {
    name     string
    chatRoom *ChatRoom
}

func NewUser(name string) *User {
    return &User{name: name}
}

func (u *User) Send(message string) {
    fmt.Printf("[%s] 送信: %s\n", u.name, message)
    u.chatRoom.SendMessage(message, u)
}

func (u *User) Receive(message, from string) {
    fmt.Printf("[%s] %s から受信: %s\n", u.name, from, message)
}

func main() {
    chatRoom := NewChatRoom()

    alice := NewUser("Alice")
    bob := NewUser("Bob")
    charlie := NewUser("Charlie")

    chatRoom.AddUser(alice)
    chatRoom.AddUser(bob)
    chatRoom.AddUser(charlie)

    alice.Send("こんにちは！")
    bob.Send("やあ、Alice！")
}
```

### イベントバスの実装

```go
package main

import (
    "fmt"
    "sync"
)

// Event はイベント
type Event struct {
    Type string
    Data any
}

// EventHandler はイベントハンドラの型
type EventHandler func(Event)

// EventBus はイベントバス（Mediator）
type EventBus struct {
    mu       sync.RWMutex
    handlers map[string][]EventHandler
}

func NewEventBus() *EventBus {
    return &EventBus{
        handlers: make(map[string][]EventHandler),
    }
}

func (b *EventBus) Subscribe(eventType string, handler EventHandler) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.handlers[eventType] = append(b.handlers[eventType], handler)
}

func (b *EventBus) Publish(event Event) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    if handlers, ok := b.handlers[event.Type]; ok {
        for _, handler := range handlers {
            handler(event)
        }
    }
}

// コンポーネント例
type UserService struct {
    bus *EventBus
}

func (s *UserService) CreateUser(name string) {
    fmt.Printf("ユーザー作成: %s\n", name)
    s.bus.Publish(Event{Type: "user.created", Data: name})
}

type EmailService struct{}

func (s *EmailService) OnUserCreated(e Event) {
    fmt.Printf("ウェルカムメール送信: %v\n", e.Data)
}

type LogService struct{}

func (s *LogService) OnAnyEvent(e Event) {
    fmt.Printf("[LOG] Event: %s, Data: %v\n", e.Type, e.Data)
}

func main() {
    bus := NewEventBus()

    userService := &UserService{bus: bus}
    emailService := &EmailService{}
    logService := &LogService{}

    // 購読
    bus.Subscribe("user.created", emailService.OnUserCreated)
    bus.Subscribe("user.created", logService.OnAnyEvent)

    // ユーザー作成（イベント発行）
    userService.CreateUser("田中太郎")
}
```

### フォームダイアログの例

```go
package main

import "fmt"

// DialogMediator はダイアログの仲介者
type DialogMediator struct {
    title    *Label
    textbox  *Textbox
    checkbox *Checkbox
    submit   *Button
}

func NewDialogMediator() *DialogMediator {
    d := &DialogMediator{}
    d.title = &Label{mediator: d, text: "タイトル"}
    d.textbox = &Textbox{mediator: d}
    d.checkbox = &Checkbox{mediator: d}
    d.submit = &Button{mediator: d, text: "送信"}
    return d
}

func (d *DialogMediator) Notify(sender any, event string) {
    switch s := sender.(type) {
    case *Textbox:
        if event == "changed" {
            d.submit.SetEnabled(len(s.text) > 0)
        }
    case *Checkbox:
        if event == "checked" {
            if s.checked {
                d.title.SetText("同意済み")
            } else {
                d.title.SetText("タイトル")
            }
        }
    case *Button:
        if event == "clicked" {
            fmt.Printf("送信: text=%s, agreed=%v\n", d.textbox.text, d.checkbox.checked)
        }
    }
}

// UIコンポーネント
type Label struct {
    mediator *DialogMediator
    text     string
}

func (l *Label) SetText(text string) {
    l.text = text
    fmt.Printf("[Label] %s\n", text)
}

type Textbox struct {
    mediator *DialogMediator
    text     string
}

func (t *Textbox) SetText(text string) {
    t.text = text
    fmt.Printf("[Textbox] 入力: %s\n", text)
    t.mediator.Notify(t, "changed")
}

type Checkbox struct {
    mediator *DialogMediator
    checked  bool
}

func (c *Checkbox) Toggle() {
    c.checked = !c.checked
    fmt.Printf("[Checkbox] チェック: %v\n", c.checked)
    c.mediator.Notify(c, "checked")
}

type Button struct {
    mediator *DialogMediator
    text     string
    enabled  bool
}

func (b *Button) SetEnabled(enabled bool) {
    b.enabled = enabled
    fmt.Printf("[Button] 有効: %v\n", enabled)
}

func (b *Button) Click() {
    if b.enabled {
        b.mediator.Notify(b, "clicked")
    }
}

func main() {
    dialog := NewDialogMediator()

    dialog.textbox.SetText("テスト入力")
    dialog.checkbox.Toggle()
    dialog.submit.Click()
}
```

## Goでのポイント

### チャネルを使ったMediator

```go
type Hub struct {
    register   chan *Client
    unregister chan *Client
    broadcast  chan []byte
    clients    map[*Client]bool
}

func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.clients[client] = true
        case client := <-h.unregister:
            delete(h.clients, client)
        case message := <-h.broadcast:
            for client := range h.clients {
                client.send <- message
            }
        }
    }
}
```

## まとめ

- Mediatorはオブジェクト間の通信を仲介し、疎結合を実現
- イベントバス、チャットルーム、UIダイアログなどで活用
- Goではチャネルと組み合わせて効果的に実装できる

次章では、オブジェクトの状態を保存・復元するMementoパターンを学びます。
