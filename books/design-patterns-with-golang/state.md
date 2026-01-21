---
title: "State パターン"
---

# State パターン

## 概要

Stateパターンは、オブジェクトの内部状態が変化したときに、オブジェクトの振る舞いを変更できるようにするパターンです。状態をオブジェクトとしてカプセル化します。

## 問題

以下のような状況で使用を検討します：

- オブジェクトの振る舞いが状態に依存する
- 状態遷移のロジックを整理したい
- 大量のif-else/switchを避けたい

## 構造

```
┌───────────────┐       ┌───────────────┐
│    Context    │──────▶│     State     │
├───────────────┤       │  (interface)  │
│ + Request()   │       ├───────────────┤
│ - state       │       │ + Handle()    │
└───────────────┘       └───────────────┘
                              △
                         ┌────┴────┐
                    ┌────────┐ ┌────────┐
                    │StateA  │ │StateB  │
                    └────────┘ └────────┘
```

## 実装

### 自動販売機の例

```go
package main

import "fmt"

// State は状態のインターフェース
type State interface {
    InsertCoin(v *VendingMachine)
    SelectProduct(v *VendingMachine)
    Dispense(v *VendingMachine)
}

// VendingMachine は自動販売機（Context）
type VendingMachine struct {
    state   State
    balance int
}

func NewVendingMachine() *VendingMachine {
    vm := &VendingMachine{}
    vm.state = &IdleState{}
    return vm
}

func (v *VendingMachine) SetState(state State) {
    v.state = state
}

func (v *VendingMachine) InsertCoin(amount int) {
    v.balance += amount
    v.state.InsertCoin(v)
}

func (v *VendingMachine) SelectProduct() {
    v.state.SelectProduct(v)
}

func (v *VendingMachine) Dispense() {
    v.state.Dispense(v)
}

// IdleState は待機状態
type IdleState struct{}

func (s *IdleState) InsertCoin(v *VendingMachine) {
    fmt.Printf("コイン投入: %d円\n", v.balance)
    v.SetState(&HasCoinState{})
}

func (s *IdleState) SelectProduct(v *VendingMachine) {
    fmt.Println("先にコインを投入してください")
}

func (s *IdleState) Dispense(v *VendingMachine) {
    fmt.Println("先にコインを投入してください")
}

// HasCoinState はコイン投入済み状態
type HasCoinState struct{}

func (s *HasCoinState) InsertCoin(v *VendingMachine) {
    fmt.Printf("追加コイン投入: 合計%d円\n", v.balance)
}

func (s *HasCoinState) SelectProduct(v *VendingMachine) {
    if v.balance >= 100 {
        fmt.Println("商品を選択しました")
        v.SetState(&DispensingState{})
        v.Dispense()
    } else {
        fmt.Println("残高不足です")
    }
}

func (s *HasCoinState) Dispense(v *VendingMachine) {
    fmt.Println("先に商品を選択してください")
}

// DispensingState は払い出し状態
type DispensingState struct{}

func (s *DispensingState) InsertCoin(v *VendingMachine) {
    fmt.Println("払い出し中です。お待ちください")
}

func (s *DispensingState) SelectProduct(v *VendingMachine) {
    fmt.Println("払い出し中です。お待ちください")
}

func (s *DispensingState) Dispense(v *VendingMachine) {
    v.balance -= 100
    fmt.Printf("商品を払い出しました。残高: %d円\n", v.balance)
    if v.balance > 0 {
        v.SetState(&HasCoinState{})
    } else {
        v.SetState(&IdleState{})
    }
}

func main() {
    vm := NewVendingMachine()

    vm.SelectProduct()   // 先にコインを投入してください
    vm.InsertCoin(50)    // コイン投入: 50円
    vm.SelectProduct()   // 残高不足です
    vm.InsertCoin(100)   // 追加コイン投入: 合計150円
    vm.SelectProduct()   // 商品を払い出しました
    vm.SelectProduct()   // 残高不足です（50円）
}
```

### TCP接続の例

```go
package main

import "fmt"

type TCPState interface {
    Open(c *TCPConnection)
    Close(c *TCPConnection)
    Send(c *TCPConnection, data string)
}

type TCPConnection struct {
    state TCPState
}

func NewTCPConnection() *TCPConnection {
    return &TCPConnection{state: &ClosedState{}}
}

func (c *TCPConnection) SetState(state TCPState) {
    c.state = state
}

func (c *TCPConnection) Open()            { c.state.Open(c) }
func (c *TCPConnection) Close()           { c.state.Close(c) }
func (c *TCPConnection) Send(data string) { c.state.Send(c, data) }

// ClosedState
type ClosedState struct{}

func (s *ClosedState) Open(c *TCPConnection) {
    fmt.Println("接続を開始します...")
    c.SetState(&OpenState{})
    fmt.Println("接続完了")
}

func (s *ClosedState) Close(c *TCPConnection) {
    fmt.Println("既に閉じています")
}

func (s *ClosedState) Send(c *TCPConnection, data string) {
    fmt.Println("接続が閉じています。先にOpenしてください")
}

// OpenState
type OpenState struct{}

func (s *OpenState) Open(c *TCPConnection) {
    fmt.Println("既に接続されています")
}

func (s *OpenState) Close(c *TCPConnection) {
    fmt.Println("接続を閉じます...")
    c.SetState(&ClosedState{})
    fmt.Println("切断完了")
}

func (s *OpenState) Send(c *TCPConnection, data string) {
    fmt.Printf("データ送信: %s\n", data)
}

func main() {
    conn := NewTCPConnection()

    conn.Send("Hello")  // 接続が閉じています
    conn.Open()         // 接続を開始します...
    conn.Send("Hello")  // データ送信: Hello
    conn.Send("World")  // データ送信: World
    conn.Close()        // 接続を閉じます...
    conn.Send("Test")   // 接続が閉じています
}
```

## Goでのポイント

### 関数型でのシンプルな実装

```go
type State func(event string) State

func IdleState(event string) State {
    switch event {
    case "start":
        return RunningState
    default:
        return IdleState
    }
}

func RunningState(event string) State {
    switch event {
    case "stop":
        return IdleState
    default:
        return RunningState
    }
}
```

## Strategyパターンとの違い

| 観点 | State | Strategy |
|-----|-------|----------|
| 目的 | 状態に応じた振る舞い変更 | アルゴリズムの交換 |
| 変更主体 | 状態オブジェクト自身が遷移 | クライアントが選択 |
| 状態の認識 | 状態同士が互いを認識 | 戦略同士は独立 |

## まとめ

- Stateは内部状態に応じてオブジェクトの振る舞いを変更する
- 状態遷移を明確にし、if-else地獄を避けられる
- Goでは構造体とインターフェースで自然に実装できる

次章では、アルゴリズムを交換可能にするStrategyパターンを学びます。
