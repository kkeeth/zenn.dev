---
title: "Observer パターン"
---

# Observer パターン

## 概要

Observerパターンは、オブジェクト間の一対多の依存関係を定義し、あるオブジェクトの状態が変化したときに、依存するすべてのオブジェクトに自動的に通知するパターンです。

## 問題

以下のような状況で使用を検討します：

- 状態の変化を複数のオブジェクトに通知したい
- イベント駆動型のシステムを構築したい
- オブジェクト間を疎結合に保ちたい

## 構造

```
┌─────────────┐        ┌─────────────┐
│   Subject   │───────▶│  Observer   │
├─────────────┤        │ (interface) │
│ + Attach()  │        ├─────────────┤
│ + Detach()  │        │ + Update()  │
│ + Notify()  │        └─────────────┘
└─────────────┘              △
                        ┌────┴────┐
                   ┌────────┐ ┌────────┐
                   │Observer│ │Observer│
                   │   A    │ │   B    │
                   └────────┘ └────────┘
```

## 実装

### 基本的な実装

```go
package main

import "fmt"

// Observer はオブザーバのインターフェース
type Observer interface {
    Update(temperature, humidity float64)
}

// Subject はサブジェクト（観察対象）
type WeatherStation struct {
    observers   []Observer
    temperature float64
    humidity    float64
}

func NewWeatherStation() *WeatherStation {
    return &WeatherStation{observers: []Observer{}}
}

func (w *WeatherStation) Attach(o Observer) {
    w.observers = append(w.observers, o)
}

func (w *WeatherStation) Detach(o Observer) {
    for i, obs := range w.observers {
        if obs == o {
            w.observers = append(w.observers[:i], w.observers[i+1:]...)
            return
        }
    }
}

func (w *WeatherStation) Notify() {
    for _, o := range w.observers {
        o.Update(w.temperature, w.humidity)
    }
}

func (w *WeatherStation) SetMeasurements(temp, humidity float64) {
    w.temperature = temp
    w.humidity = humidity
    w.Notify()
}

// PhoneDisplay はスマホ表示（具体的なObserver）
type PhoneDisplay struct {
    name string
}

func (p *PhoneDisplay) Update(temp, humidity float64) {
    fmt.Printf("[%s] 気温: %.1f°C, 湿度: %.1f%%\n", p.name, temp, humidity)
}

// WebDisplay はWeb表示
type WebDisplay struct{}

func (w *WebDisplay) Update(temp, humidity float64) {
    fmt.Printf("[Web] Temperature: %.1f°C | Humidity: %.1f%%\n", temp, humidity)
}

func main() {
    station := NewWeatherStation()

    phone := &PhoneDisplay{name: "iPhone"}
    web := &WebDisplay{}

    station.Attach(phone)
    station.Attach(web)

    fmt.Println("=== 測定1 ===")
    station.SetMeasurements(25.5, 60.0)

    fmt.Println("\n=== 測定2 ===")
    station.SetMeasurements(28.0, 55.0)

    station.Detach(phone)
    fmt.Println("\n=== 測定3（phone削除後）===")
    station.SetMeasurements(22.0, 70.0)
}
```

### チャネルを使った実装（Goらしい実装）

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

// EventEmitter はイベント発行者
type EventEmitter struct {
    mu        sync.RWMutex
    listeners map[string][]chan Event
}

func NewEventEmitter() *EventEmitter {
    return &EventEmitter{
        listeners: make(map[string][]chan Event),
    }
}

func (e *EventEmitter) On(eventType string) <-chan Event {
    e.mu.Lock()
    defer e.mu.Unlock()

    ch := make(chan Event, 10)
    e.listeners[eventType] = append(e.listeners[eventType], ch)
    return ch
}

func (e *EventEmitter) Emit(event Event) {
    e.mu.RLock()
    defer e.mu.RUnlock()

    if listeners, ok := e.listeners[event.Type]; ok {
        for _, ch := range listeners {
            select {
            case ch <- event:
            default:
                // バッファがいっぱいの場合はスキップ
            }
        }
    }
}

func (e *EventEmitter) Close() {
    e.mu.Lock()
    defer e.mu.Unlock()

    for _, listeners := range e.listeners {
        for _, ch := range listeners {
            close(ch)
        }
    }
}

func main() {
    emitter := NewEventEmitter()

    // リスナー1
    ch1 := emitter.On("message")
    go func() {
        for event := range ch1 {
            fmt.Printf("[Listener1] Received: %v\n", event.Data)
        }
    }()

    // リスナー2
    ch2 := emitter.On("message")
    go func() {
        for event := range ch2 {
            fmt.Printf("[Listener2] Got message: %v\n", event.Data)
        }
    }()

    // イベント発行
    emitter.Emit(Event{Type: "message", Data: "Hello"})
    emitter.Emit(Event{Type: "message", Data: "World"})

    // 少し待機
    fmt.Scanln()
}
```

### コールバック関数を使った実装

```go
package main

import "fmt"

type Callback func(data any)

type Subject struct {
    callbacks []Callback
}

func (s *Subject) Subscribe(cb Callback) {
    s.callbacks = append(s.callbacks, cb)
}

func (s *Subject) Notify(data any) {
    for _, cb := range s.callbacks {
        cb(data)
    }
}

// 使用例
type Stock struct {
    Subject
    symbol string
    price  float64
}

func (s *Stock) SetPrice(price float64) {
    s.price = price
    s.Notify(map[string]any{
        "symbol": s.symbol,
        "price":  price,
    })
}

func main() {
    stock := &Stock{symbol: "GOOG"}

    stock.Subscribe(func(data any) {
        d := data.(map[string]any)
        fmt.Printf("[Alert] %s: $%.2f\n", d["symbol"], d["price"])
    })

    stock.Subscribe(func(data any) {
        d := data.(map[string]any)
        if d["price"].(float64) > 150 {
            fmt.Printf("[Warning] %s is expensive!\n", d["symbol"])
        }
    })

    stock.SetPrice(145.00)
    stock.SetPrice(155.00)
}
```

## Goでのポイント

### context.Contextとの組み合わせ

```go
func Subscribe(ctx context.Context, events <-chan Event) {
    for {
        select {
        case <-ctx.Done():
            return
        case event := <-events:
            handleEvent(event)
        }
    }
}
```

### sync.Condの活用

```go
var cond = sync.NewCond(&sync.Mutex{})

func Wait() {
    cond.L.Lock()
    cond.Wait()
    cond.L.Unlock()
}

func Signal() {
    cond.Broadcast()
}
```

## まとめ

- Observerは状態変化を複数のオブジェクトに通知する
- Goではチャネルやコールバックで自然に実装できる
- イベント駆動システムの基盤パターン

次章では、状態に応じて振る舞いを変更するStateパターンを学びます。
