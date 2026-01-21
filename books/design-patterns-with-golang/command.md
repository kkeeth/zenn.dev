---
title: "Command パターン"
---

# Command パターン

## 概要

Commandパターンは、リクエストをオブジェクトとしてカプセル化するパターンです。これにより、リクエストのパラメータ化、キュー登録、ログ記録、取り消し操作などが可能になります。

## 問題

以下のような状況で使用を検討します：

- 操作の実行を遅延させたい
- 操作の取り消し（Undo）/やり直し（Redo）を実装したい
- 操作をキューに登録して順次実行したい
- 操作のログを記録したい

## 構造

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Invoker   │────▶│   Command   │────▶│  Receiver   │
├─────────────┤     │ (interface) │     ├─────────────┤
│ + Execute() │     ├─────────────┤     │ + Action()  │
└─────────────┘     │ + Execute() │     └─────────────┘
                    │ + Undo()    │
                    └─────────────┘
```

## 実装

### テキストエディタの例（Undo/Redo対応）

```go
package main

import "fmt"

// Command はコマンドのインターフェース
type Command interface {
    Execute()
    Undo()
}

// Editor はテキストエディタ（Receiver）
type Editor struct {
    text string
}

func (e *Editor) GetText() string {
    return e.text
}

func (e *Editor) SetText(text string) {
    e.text = text
}

// InsertCommand は文字挿入コマンド
type InsertCommand struct {
    editor   *Editor
    position int
    text     string
}

func NewInsertCommand(editor *Editor, position int, text string) *InsertCommand {
    return &InsertCommand{editor: editor, position: position, text: text}
}

func (c *InsertCommand) Execute() {
    current := c.editor.GetText()
    newText := current[:c.position] + c.text + current[c.position:]
    c.editor.SetText(newText)
}

func (c *InsertCommand) Undo() {
    current := c.editor.GetText()
    newText := current[:c.position] + current[c.position+len(c.text):]
    c.editor.SetText(newText)
}

// DeleteCommand は文字削除コマンド
type DeleteCommand struct {
    editor      *Editor
    position    int
    length      int
    deletedText string
}

func NewDeleteCommand(editor *Editor, position, length int) *DeleteCommand {
    return &DeleteCommand{editor: editor, position: position, length: length}
}

func (c *DeleteCommand) Execute() {
    current := c.editor.GetText()
    c.deletedText = current[c.position : c.position+c.length]
    newText := current[:c.position] + current[c.position+c.length:]
    c.editor.SetText(newText)
}

func (c *DeleteCommand) Undo() {
    current := c.editor.GetText()
    newText := current[:c.position] + c.deletedText + current[c.position:]
    c.editor.SetText(newText)
}

// CommandHistory はコマンド履歴を管理
type CommandHistory struct {
    history []Command
    current int
}

func NewCommandHistory() *CommandHistory {
    return &CommandHistory{history: []Command{}, current: -1}
}

func (h *CommandHistory) Execute(cmd Command) {
    // 現在位置以降の履歴を削除
    h.history = h.history[:h.current+1]
    cmd.Execute()
    h.history = append(h.history, cmd)
    h.current++
}

func (h *CommandHistory) Undo() bool {
    if h.current < 0 {
        return false
    }
    h.history[h.current].Undo()
    h.current--
    return true
}

func (h *CommandHistory) Redo() bool {
    if h.current >= len(h.history)-1 {
        return false
    }
    h.current++
    h.history[h.current].Execute()
    return true
}

func main() {
    editor := &Editor{}
    history := NewCommandHistory()

    // テキストを入力
    history.Execute(NewInsertCommand(editor, 0, "Hello"))
    fmt.Printf("After insert 'Hello': '%s'\n", editor.GetText())

    history.Execute(NewInsertCommand(editor, 5, " World"))
    fmt.Printf("After insert ' World': '%s'\n", editor.GetText())

    history.Execute(NewInsertCommand(editor, 11, "!"))
    fmt.Printf("After insert '!': '%s'\n", editor.GetText())

    // Undo
    history.Undo()
    fmt.Printf("After Undo: '%s'\n", editor.GetText())

    history.Undo()
    fmt.Printf("After Undo: '%s'\n", editor.GetText())

    // Redo
    history.Redo()
    fmt.Printf("After Redo: '%s'\n", editor.GetText())
}
```

### 関数型での実装（Goでシンプル）

```go
package main

import "fmt"

// Command は関数型のコマンド
type Command func()

// CommandQueue はコマンドキュー
type CommandQueue struct {
    queue []Command
}

func (q *CommandQueue) Add(cmd Command) {
    q.queue = append(q.queue, cmd)
}

func (q *CommandQueue) ExecuteAll() {
    for _, cmd := range q.queue {
        cmd()
    }
    q.queue = nil
}

// Light はライト（Receiver）
type Light struct {
    isOn bool
    name string
}

func (l *Light) On() {
    l.isOn = true
    fmt.Printf("%s: ON\n", l.name)
}

func (l *Light) Off() {
    l.isOn = false
    fmt.Printf("%s: OFF\n", l.name)
}

func main() {
    livingRoom := &Light{name: "リビングの照明"}
    kitchen := &Light{name: "キッチンの照明"}

    queue := &CommandQueue{}

    // コマンドをキューに追加
    queue.Add(livingRoom.On)
    queue.Add(kitchen.On)
    queue.Add(func() {
        fmt.Println("1秒待機...")
    })
    queue.Add(kitchen.Off)

    // 一括実行
    fmt.Println("=== コマンドを実行 ===")
    queue.ExecuteAll()
}
```

### リモコンの例

```go
package main

import "fmt"

type Command interface {
    Execute()
}

type TV struct {
    isOn bool
}

func (t *TV) On()  { t.isOn = true; fmt.Println("TV ON") }
func (t *TV) Off() { t.isOn = false; fmt.Println("TV OFF") }

type TVOnCommand struct{ tv *TV }
func (c *TVOnCommand) Execute() { c.tv.On() }

type TVOffCommand struct{ tv *TV }
func (c *TVOffCommand) Execute() { c.tv.Off() }

type RemoteControl struct {
    commands map[string]Command
}

func NewRemoteControl() *RemoteControl {
    return &RemoteControl{commands: make(map[string]Command)}
}

func (r *RemoteControl) SetCommand(button string, cmd Command) {
    r.commands[button] = cmd
}

func (r *RemoteControl) PressButton(button string) {
    if cmd, ok := r.commands[button]; ok {
        cmd.Execute()
    }
}

func main() {
    tv := &TV{}
    remote := NewRemoteControl()

    remote.SetCommand("1", &TVOnCommand{tv: tv})
    remote.SetCommand("2", &TVOffCommand{tv: tv})

    remote.PressButton("1")
    remote.PressButton("2")
}
```

## Goでのポイント

### クロージャの活用

```go
func MakeCommand(receiver *Receiver, action func(*Receiver)) Command {
    return func() {
        action(receiver)
    }
}
```

### チャネルとの組み合わせ

```go
func Worker(commands <-chan Command) {
    for cmd := range commands {
        cmd.Execute()
    }
}
```

## まとめ

- Commandは操作をオブジェクト（または関数）としてカプセル化する
- Undo/Redo、キュー、ログ記録などに活用できる
- Goでは関数型でシンプルに実装可能

次章では、言語の文法を解釈するInterpreterパターンを学びます。
