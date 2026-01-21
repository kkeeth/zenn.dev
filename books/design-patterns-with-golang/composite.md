---
title: "Composite パターン"
---

# Composite パターン

## 概要

Compositeパターンは、オブジェクトを木構造で構成し、個別のオブジェクトとオブジェクトの集合を同一視して扱えるようにするパターンです。

## 問題

以下のような状況で使用を検討します：

- 部分-全体の階層構造を表現したい
- クライアントが個別要素と複合要素を同じように扱いたい
- ファイルシステム、組織構造、GUI部品などの木構造を実装したい

## 構造

```
┌─────────────────┐
│   Component     │
│  (interface)    │
├─────────────────┤
│ + Operation()   │
└─────────────────┘
        △
        │
   ┌────┴────┐
   │         │
┌──────┐  ┌──────────┐
│ Leaf │  │Composite │
└──────┘  ├──────────┤
          │-children │
          └──────────┘
```

## 実装

### ファイルシステムの例

```go
package main

import (
    "fmt"
    "strings"
)

// Component はファイルシステムの要素を表すインターフェース
type FileSystemComponent interface {
    GetName() string
    GetSize() int64
    Display(indent string)
}

// File はファイル（Leaf）
type File struct {
    name string
    size int64
}

func NewFile(name string, size int64) *File {
    return &File{name: name, size: size}
}

func (f *File) GetName() string { return f.name }
func (f *File) GetSize() int64  { return f.size }
func (f *File) Display(indent string) {
    fmt.Printf("%s📄 %s (%d bytes)\n", indent, f.name, f.size)
}

// Directory はディレクトリ（Composite）
type Directory struct {
    name     string
    children []FileSystemComponent
}

func NewDirectory(name string) *Directory {
    return &Directory{name: name, children: []FileSystemComponent{}}
}

func (d *Directory) GetName() string { return d.name }

func (d *Directory) GetSize() int64 {
    var total int64
    for _, child := range d.children {
        total += child.GetSize()
    }
    return total
}

func (d *Directory) Display(indent string) {
    fmt.Printf("%s📁 %s/ (%d bytes)\n", indent, d.name, d.GetSize())
    for _, child := range d.children {
        child.Display(indent + "  ")
    }
}

func (d *Directory) Add(component FileSystemComponent) {
    d.children = append(d.children, component)
}

func (d *Directory) Remove(name string) {
    for i, child := range d.children {
        if child.GetName() == name {
            d.children = append(d.children[:i], d.children[i+1:]...)
            return
        }
    }
}

func main() {
    // ファイルシステム構造を構築
    root := NewDirectory("root")

    // documentsディレクトリ
    docs := NewDirectory("documents")
    docs.Add(NewFile("readme.txt", 1024))
    docs.Add(NewFile("report.pdf", 2048))

    // picturesディレクトリ
    pictures := NewDirectory("pictures")
    pictures.Add(NewFile("photo1.jpg", 3072))
    pictures.Add(NewFile("photo2.jpg", 4096))

    // 2024サブディレクトリ
    pics2024 := NewDirectory("2024")
    pics2024.Add(NewFile("newyear.jpg", 5120))
    pictures.Add(pics2024)

    root.Add(docs)
    root.Add(pictures)
    root.Add(NewFile("config.yaml", 512))

    // 表示
    root.Display("")
}
```

### 組織構造の例

```go
package main

import "fmt"

// Employee は社員を表すインターフェース
type Employee interface {
    GetName() string
    GetSalary() int
    GetRole() string
    Display(indent string)
}

// Developer は開発者（Leaf）
type Developer struct {
    name   string
    salary int
}

func NewDeveloper(name string, salary int) *Developer {
    return &Developer{name: name, salary: salary}
}

func (d *Developer) GetName() string   { return d.name }
func (d *Developer) GetSalary() int    { return d.salary }
func (d *Developer) GetRole() string   { return "Developer" }
func (d *Developer) Display(indent string) {
    fmt.Printf("%s👨‍💻 %s (%s) - ¥%d\n", indent, d.name, d.GetRole(), d.salary)
}

// Designer はデザイナー（Leaf）
type Designer struct {
    name   string
    salary int
}

func NewDesigner(name string, salary int) *Designer {
    return &Designer{name: name, salary: salary}
}

func (d *Designer) GetName() string   { return d.name }
func (d *Designer) GetSalary() int    { return d.salary }
func (d *Designer) GetRole() string   { return "Designer" }
func (d *Designer) Display(indent string) {
    fmt.Printf("%s🎨 %s (%s) - ¥%d\n", indent, d.name, d.GetRole(), d.salary)
}

// Manager はマネージャー（Composite）
type Manager struct {
    name        string
    salary      int
    subordinates []Employee
}

func NewManager(name string, salary int) *Manager {
    return &Manager{name: name, salary: salary, subordinates: []Employee{}}
}

func (m *Manager) GetName() string { return m.name }

func (m *Manager) GetSalary() int {
    total := m.salary
    for _, sub := range m.subordinates {
        total += sub.GetSalary()
    }
    return total
}

func (m *Manager) GetRole() string { return "Manager" }

func (m *Manager) Display(indent string) {
    fmt.Printf("%s👔 %s (%s) - ¥%d (チーム合計: ¥%d)\n",
        indent, m.name, m.GetRole(), m.salary, m.GetSalary())
    for _, sub := range m.subordinates {
        sub.Display(indent + "  ")
    }
}

func (m *Manager) Add(employee Employee) {
    m.subordinates = append(m.subordinates, employee)
}

func main() {
    // 組織構造を構築
    ceo := NewManager("佐藤社長", 10000000)

    // 開発部門
    devManager := NewManager("田中部長", 5000000)
    devManager.Add(NewDeveloper("山田", 4000000))
    devManager.Add(NewDeveloper("鈴木", 3500000))

    // デザイン部門
    designManager := NewManager("高橋部長", 5000000)
    designManager.Add(NewDesigner("伊藤", 4000000))
    designManager.Add(NewDesigner("渡辺", 3500000))

    ceo.Add(devManager)
    ceo.Add(designManager)

    fmt.Println("=== 組織図 ===")
    ceo.Display("")

    fmt.Printf("\n会社全体の人件費: ¥%d\n", ceo.GetSalary())
}
```

## 実践的な例：UIコンポーネント

```go
package main

import (
    "fmt"
    "strings"
)

// UIComponent はUI要素を表すインターフェース
type UIComponent interface {
    Render() string
    AddClass(class string)
    GetClasses() []string
}

// BaseComponent は共通の基底構造
type BaseComponent struct {
    classes []string
}

func (b *BaseComponent) AddClass(class string) {
    b.classes = append(b.classes, class)
}

func (b *BaseComponent) GetClasses() []string {
    return b.classes
}

func (b *BaseComponent) classAttr() string {
    if len(b.classes) == 0 {
        return ""
    }
    return fmt.Sprintf(` class="%s"`, strings.Join(b.classes, " "))
}

// Button はボタン（Leaf）
type Button struct {
    BaseComponent
    text string
}

func NewButton(text string) *Button {
    return &Button{text: text}
}

func (b *Button) Render() string {
    return fmt.Sprintf("<button%s>%s</button>", b.classAttr(), b.text)
}

// Input は入力フィールド（Leaf）
type Input struct {
    BaseComponent
    inputType   string
    placeholder string
}

func NewInput(inputType, placeholder string) *Input {
    return &Input{inputType: inputType, placeholder: placeholder}
}

func (i *Input) Render() string {
    return fmt.Sprintf(`<input type="%s" placeholder="%s"%s />`,
        i.inputType, i.placeholder, i.classAttr())
}

// Text はテキスト（Leaf）
type Text struct {
    BaseComponent
    content string
}

func NewText(content string) *Text {
    return &Text{content: content}
}

func (t *Text) Render() string {
    return fmt.Sprintf("<span%s>%s</span>", t.classAttr(), t.content)
}

// Container はコンテナ（Composite）
type Container struct {
    BaseComponent
    tag      string
    children []UIComponent
}

func NewContainer(tag string) *Container {
    return &Container{tag: tag, children: []UIComponent{}}
}

func NewDiv() *Container {
    return NewContainer("div")
}

func NewForm() *Container {
    return NewContainer("form")
}

func (c *Container) Add(component UIComponent) *Container {
    c.children = append(c.children, component)
    return c
}

func (c *Container) Render() string {
    var sb strings.Builder
    sb.WriteString(fmt.Sprintf("<%s%s>\n", c.tag, c.classAttr()))
    for _, child := range c.children {
        sb.WriteString("  ")
        sb.WriteString(child.Render())
        sb.WriteString("\n")
    }
    sb.WriteString(fmt.Sprintf("</%s>", c.tag))
    return sb.String()
}

func main() {
    // ログインフォームを構築
    form := NewForm()
    form.AddClass("login-form")

    usernameGroup := NewDiv()
    usernameGroup.AddClass("form-group")
    usernameGroup.Add(NewText("ユーザー名:"))
    usernameInput := NewInput("text", "ユーザー名を入力")
    usernameInput.AddClass("form-control")
    usernameGroup.Add(usernameInput)

    passwordGroup := NewDiv()
    passwordGroup.AddClass("form-group")
    passwordGroup.Add(NewText("パスワード:"))
    passwordInput := NewInput("password", "パスワードを入力")
    passwordInput.AddClass("form-control")
    passwordGroup.Add(passwordInput)

    buttonGroup := NewDiv()
    buttonGroup.AddClass("button-group")
    submitBtn := NewButton("ログイン")
    submitBtn.AddClass("btn")
    submitBtn.AddClass("btn-primary")
    buttonGroup.Add(submitBtn)

    form.Add(usernameGroup)
    form.Add(passwordGroup)
    form.Add(buttonGroup)

    fmt.Println(form.Render())
}
```

## 実践的な例：メニュー構造

```go
package main

import "fmt"

// MenuComponent はメニュー要素のインターフェース
type MenuComponent interface {
    Print(indent int)
    GetPrice() float64
}

// MenuItem はメニュー項目（Leaf）
type MenuItem struct {
    name        string
    description string
    price       float64
}

func NewMenuItem(name, description string, price float64) *MenuItem {
    return &MenuItem{name: name, description: description, price: price}
}

func (m *MenuItem) Print(indent int) {
    prefix := ""
    for i := 0; i < indent; i++ {
        prefix += "  "
    }
    fmt.Printf("%s%s (¥%.0f)\n", prefix, m.name, m.price)
    fmt.Printf("%s  %s\n", prefix, m.description)
}

func (m *MenuItem) GetPrice() float64 {
    return m.price
}

// Menu はメニューカテゴリ（Composite）
type Menu struct {
    name       string
    components []MenuComponent
}

func NewMenu(name string) *Menu {
    return &Menu{name: name, components: []MenuComponent{}}
}

func (m *Menu) Add(component MenuComponent) {
    m.components = append(m.components, component)
}

func (m *Menu) Print(indent int) {
    prefix := ""
    for i := 0; i < indent; i++ {
        prefix += "  "
    }
    fmt.Printf("%s【%s】\n", prefix, m.name)
    for _, c := range m.components {
        c.Print(indent + 1)
    }
}

func (m *Menu) GetPrice() float64 {
    var total float64
    for _, c := range m.components {
        total += c.GetPrice()
    }
    return total
}

func main() {
    // メインメニュー
    mainMenu := NewMenu("メインメニュー")

    // ランチメニュー
    lunchMenu := NewMenu("ランチメニュー")
    lunchMenu.Add(NewMenuItem("日替わり定食", "本日の主菜と副菜2品", 800))
    lunchMenu.Add(NewMenuItem("カレーライス", "スパイシーな特製カレー", 700))

    // ディナーメニュー
    dinnerMenu := NewMenu("ディナーメニュー")
    dinnerMenu.Add(NewMenuItem("ステーキセット", "200g和牛ステーキ", 2500))

    // ドリンクサブメニュー
    drinkMenu := NewMenu("ドリンク")
    drinkMenu.Add(NewMenuItem("コーヒー", "ブレンドコーヒー", 300))
    drinkMenu.Add(NewMenuItem("紅茶", "ダージリン", 350))
    dinnerMenu.Add(drinkMenu)

    mainMenu.Add(lunchMenu)
    mainMenu.Add(dinnerMenu)

    mainMenu.Print(0)
}
```

## Goでのポイント

### インターフェースの定義

再帰的な構造ではインターフェースが重要です：

```go
type Component interface {
    Operation()
}

type Composite struct {
    children []Component // インターフェース型のスライス
}
```

### 安全な型アサーション

Compositeかどうかを判定する場合：

```go
func AddChild(parent, child Component) bool {
    if composite, ok := parent.(*Composite); ok {
        composite.Add(child)
        return true
    }
    return false
}
```

## まとめ

- Compositeは部分-全体の階層構造を統一的に扱う
- Leafと Compositeが同じインターフェースを実装
- ファイルシステム、組織構造、UIなど木構造の実装に有効

次章では、動的に機能を追加するDecoratorパターンを学びます。
