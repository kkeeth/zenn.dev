---
title: "Memento パターン"
---

# Memento パターン

## 概要

Mementoパターンは、オブジェクトの内部状態をカプセル化して保存し、後で復元できるようにするパターンです。Undo機能やチェックポイントの実装に使用されます。

## 問題

以下のような状況で使用を検討します：

- オブジェクトの状態を保存・復元したい
- Undo/Redo機能を実装したい
- スナップショットを作成したい

## 構造

```
┌────────────┐     ┌────────────┐     ┌────────────┐
│ Originator │────▶│  Memento   │◀────│ Caretaker  │
├────────────┤     ├────────────┤     ├────────────┤
│ + Save()   │     │ - state    │     │ - mementos │
│ + Restore()│     └────────────┘     └────────────┘
└────────────┘
```

## 実装

### テキストエディタの例

```go
package main

import "fmt"

// Memento は状態を保存するスナップショット
type Memento struct {
    state string
}

func (m *Memento) GetState() string {
    return m.state
}

// Editor はテキストエディタ（Originator）
type Editor struct {
    text string
}

func (e *Editor) SetText(text string) {
    e.text = text
}

func (e *Editor) GetText() string {
    return e.text
}

func (e *Editor) Save() *Memento {
    return &Memento{state: e.text}
}

func (e *Editor) Restore(m *Memento) {
    e.text = m.GetState()
}

// History は履歴管理（Caretaker）
type History struct {
    mementos []*Memento
}

func NewHistory() *History {
    return &History{mementos: []*Memento{}}
}

func (h *History) Push(m *Memento) {
    h.mementos = append(h.mementos, m)
}

func (h *History) Pop() *Memento {
    if len(h.mementos) == 0 {
        return nil
    }
    last := h.mementos[len(h.mementos)-1]
    h.mementos = h.mementos[:len(h.mementos)-1]
    return last
}

func main() {
    editor := &Editor{}
    history := NewHistory()

    editor.SetText("Hello")
    history.Push(editor.Save())
    fmt.Printf("状態1: %s\n", editor.GetText())

    editor.SetText("Hello World")
    history.Push(editor.Save())
    fmt.Printf("状態2: %s\n", editor.GetText())

    editor.SetText("Hello World!")
    fmt.Printf("状態3: %s\n", editor.GetText())

    // Undo
    editor.Restore(history.Pop())
    fmt.Printf("Undo後: %s\n", editor.GetText())

    editor.Restore(history.Pop())
    fmt.Printf("Undo後: %s\n", editor.GetText())
}
```

### ゲームのセーブデータ

```go
package main

import (
    "encoding/json"
    "fmt"
)

// GameState はゲーム状態のMemento
type GameState struct {
    Level      int    `json:"level"`
    Score      int    `json:"score"`
    PlayerX    int    `json:"player_x"`
    PlayerY    int    `json:"player_y"`
    Inventory  []string `json:"inventory"`
    SavedAt    string `json:"saved_at"`
}

// Game はゲーム（Originator）
type Game struct {
    level     int
    score     int
    playerX   int
    playerY   int
    inventory []string
}

func NewGame() *Game {
    return &Game{
        level:     1,
        inventory: []string{},
    }
}

func (g *Game) Play() {
    g.score += 100
    g.playerX += 10
    g.playerY += 5
    fmt.Printf("プレイ中... Score: %d, Position: (%d, %d)\n",
        g.score, g.playerX, g.playerY)
}

func (g *Game) CollectItem(item string) {
    g.inventory = append(g.inventory, item)
    fmt.Printf("アイテム取得: %s\n", item)
}

func (g *Game) LevelUp() {
    g.level++
    fmt.Printf("レベルアップ！ Level: %d\n", g.level)
}

func (g *Game) Save() *GameState {
    inv := make([]string, len(g.inventory))
    copy(inv, g.inventory)
    return &GameState{
        Level:     g.level,
        Score:     g.score,
        PlayerX:   g.playerX,
        PlayerY:   g.playerY,
        Inventory: inv,
    }
}

func (g *Game) Load(state *GameState) {
    g.level = state.Level
    g.score = state.Score
    g.playerX = state.PlayerX
    g.playerY = state.PlayerY
    g.inventory = make([]string, len(state.Inventory))
    copy(g.inventory, state.Inventory)
}

func (g *Game) Status() {
    fmt.Printf("Level: %d, Score: %d, Position: (%d, %d), Items: %v\n",
        g.level, g.score, g.playerX, g.playerY, g.inventory)
}

// SaveManager はセーブデータ管理（Caretaker）
type SaveManager struct {
    slots map[int]*GameState
}

func NewSaveManager() *SaveManager {
    return &SaveManager{slots: make(map[int]*GameState)}
}

func (m *SaveManager) SaveToSlot(slot int, state *GameState) {
    m.slots[slot] = state
    fmt.Printf("スロット %d にセーブしました\n", slot)
}

func (m *SaveManager) LoadFromSlot(slot int) *GameState {
    return m.slots[slot]
}

func main() {
    game := NewGame()
    saveManager := NewSaveManager()

    // ゲームプレイ
    game.Play()
    game.CollectItem("剣")

    // セーブ
    saveManager.SaveToSlot(1, game.Save())

    // さらにプレイ
    game.Play()
    game.CollectItem("盾")
    game.LevelUp()

    fmt.Println("\n現在の状態:")
    game.Status()

    // ロード
    fmt.Println("\nスロット1からロード:")
    game.Load(saveManager.LoadFromSlot(1))
    game.Status()
}
```

### JSONシリアライズでの実装

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Config struct {
    Server   string
    Port     int
    Debug    bool
    Features map[string]bool
}

func (c *Config) Snapshot() ([]byte, error) {
    return json.Marshal(c)
}

func (c *Config) Restore(data []byte) error {
    return json.Unmarshal(data, c)
}

func main() {
    config := &Config{
        Server:   "localhost",
        Port:     8080,
        Debug:    false,
        Features: map[string]bool{"cache": true},
    }

    // スナップショット作成
    snapshot, _ := config.Snapshot()
    fmt.Printf("保存: %s\n", snapshot)

    // 設定変更
    config.Debug = true
    config.Port = 9090
    fmt.Printf("変更後: Debug=%v, Port=%d\n", config.Debug, config.Port)

    // 復元
    config.Restore(snapshot)
    fmt.Printf("復元後: Debug=%v, Port=%d\n", config.Debug, config.Port)
}
```

## Goでのポイント

### 値のコピーで実現

```go
type State struct {
    Data int
}

// コピーを返すことでMementoを実現
func (s *State) Snapshot() State {
    return *s // 値コピー
}

func (s *State) Restore(snapshot State) {
    *s = snapshot
}
```

### 深いコピーに注意

スライスやマップを含む場合は深いコピーが必要です。

## まとめ

- Mementoはオブジェクトの状態を保存・復元する
- Undo機能、セーブ機能、チェックポイントに活用
- Goでは値コピーやJSONシリアライズで実装可能

次章では、状態変化を通知するObserverパターンを学びます。
