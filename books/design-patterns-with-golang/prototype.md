---
title: "Prototype パターン"
---

# Prototype パターン

## 概要

Prototypeパターンは、既存のオブジェクト（プロトタイプ）をコピーして新しいオブジェクトを生成するパターンです。オブジェクトの生成コストが高い場合や、実行時に生成するオブジェクトの種類を決定したい場合に有効です。

## 問題

以下のような状況で使用を検討します：

- オブジェクトの生成コストが高い（DB接続、ファイル読み込み、複雑な計算など）
- 生成するオブジェクトの種類を実行時に決定したい
- 類似したオブジェクトを大量に生成したい
- 具体的なクラスに依存せずにオブジェクトを生成したい

## 構造

```
┌─────────────────────┐
│     Prototype       │
│    (interface)      │
├─────────────────────┤
│ + Clone(): Prototype│
└─────────────────────┘
          △
          │
    ┌─────┴─────┐
    │           │
┌───────┐   ┌───────┐
│Proto1 │   │Proto2 │
│Clone()│   │Clone()│
└───────┘   └───────┘
```

## 実装

### 基本的な実装

```go
package main

import "fmt"

// Cloneable はクローン可能なオブジェクトのインターフェース
type Cloneable interface {
    Clone() Cloneable
}

// Document はドキュメントを表す
type Document struct {
    Title   string
    Content string
    Author  string
}

// Clone はDocumentのディープコピーを返す
func (d *Document) Clone() Cloneable {
    return &Document{
        Title:   d.Title,
        Content: d.Content,
        Author:  d.Author,
    }
}

func (d *Document) String() string {
    return fmt.Sprintf("Document{Title=%s, Author=%s}", d.Title, d.Author)
}

func main() {
    // 元のドキュメント
    original := &Document{
        Title:   "デザインパターン入門",
        Content: "本書ではGoでデザインパターンを学びます...",
        Author:  "田中太郎",
    }

    // クローンを作成
    cloned := original.Clone().(*Document)

    // クローンを変更しても元のオブジェクトには影響しない
    cloned.Title = "デザインパターン入門 改訂版"
    cloned.Author = "田中太郎, 山田花子"

    fmt.Println("Original:", original)
    fmt.Println("Cloned:", cloned)
}
```

### シャローコピーとディープコピー

```go
package main

import "fmt"

// Address は住所を表す
type Address struct {
    City   string
    Street string
}

// Person は人を表す
type Person struct {
    Name    string
    Age     int
    Address *Address // ポインタフィールド
}

// ShallowClone はシャローコピーを返す（ポインタは共有される）
func (p *Person) ShallowClone() *Person {
    return &Person{
        Name:    p.Name,
        Age:     p.Age,
        Address: p.Address, // 同じアドレスを参照
    }
}

// DeepClone はディープコピーを返す（ポインタもコピーされる）
func (p *Person) DeepClone() *Person {
    return &Person{
        Name: p.Name,
        Age:  p.Age,
        Address: &Address{ // 新しいAddressを作成
            City:   p.Address.City,
            Street: p.Address.Street,
        },
    }
}

func main() {
    original := &Person{
        Name: "田中太郎",
        Age:  30,
        Address: &Address{
            City:   "東京",
            Street: "渋谷1-1-1",
        },
    }

    // シャローコピー
    shallow := original.ShallowClone()
    shallow.Address.City = "大阪" // 元のオブジェクトにも影響する！

    fmt.Printf("Original Address: %s\n", original.Address.City) // 大阪

    // ディープコピー
    original.Address.City = "東京" // 元に戻す
    deep := original.DeepClone()
    deep.Address.City = "福岡" // 元のオブジェクトには影響しない

    fmt.Printf("Original Address: %s\n", original.Address.City) // 東京
    fmt.Printf("Deep Clone Address: %s\n", deep.Address.City)   // 福岡
}
```

### プロトタイプレジストリ

```go
package main

import "fmt"

// Shape は図形を表すインターフェース
type Shape interface {
    Clone() Shape
    GetInfo() string
}

// Rectangle は長方形
type Rectangle struct {
    Width  int
    Height int
    Color  string
}

func (r *Rectangle) Clone() Shape {
    return &Rectangle{
        Width:  r.Width,
        Height: r.Height,
        Color:  r.Color,
    }
}

func (r *Rectangle) GetInfo() string {
    return fmt.Sprintf("Rectangle{%dx%d, %s}", r.Width, r.Height, r.Color)
}

// Circle は円
type Circle struct {
    Radius int
    Color  string
}

func (c *Circle) Clone() Shape {
    return &Circle{
        Radius: c.Radius,
        Color:  c.Color,
    }
}

func (c *Circle) GetInfo() string {
    return fmt.Sprintf("Circle{r=%d, %s}", c.Radius, c.Color)
}

// ShapeRegistry はプロトタイプを管理するレジストリ
type ShapeRegistry struct {
    prototypes map[string]Shape
}

func NewShapeRegistry() *ShapeRegistry {
    return &ShapeRegistry{
        prototypes: make(map[string]Shape),
    }
}

// Register はプロトタイプを登録する
func (r *ShapeRegistry) Register(name string, shape Shape) {
    r.prototypes[name] = shape
}

// Get はプロトタイプのクローンを返す
func (r *ShapeRegistry) Get(name string) (Shape, error) {
    if proto, ok := r.prototypes[name]; ok {
        return proto.Clone(), nil
    }
    return nil, fmt.Errorf("prototype not found: %s", name)
}

func main() {
    registry := NewShapeRegistry()

    // プロトタイプを登録
    registry.Register("red-rectangle", &Rectangle{Width: 100, Height: 50, Color: "red"})
    registry.Register("blue-circle", &Circle{Radius: 30, Color: "blue"})
    registry.Register("large-square", &Rectangle{Width: 200, Height: 200, Color: "green"})

    // クローンを生成
    shape1, _ := registry.Get("red-rectangle")
    shape2, _ := registry.Get("blue-circle")
    shape3, _ := registry.Get("red-rectangle")

    fmt.Println(shape1.GetInfo())
    fmt.Println(shape2.GetInfo())
    fmt.Println(shape3.GetInfo())

    // shape1とshape3は同じプロトタイプからのクローンだが、別のインスタンス
    fmt.Println(shape1 == shape3) // false
}
```

## 実践的な例：設定テンプレート

```go
package main

import (
    "encoding/json"
    "fmt"
)

// Config はアプリケーション設定
type Config struct {
    AppName     string            `json:"app_name"`
    Environment string            `json:"environment"`
    Database    DatabaseConfig    `json:"database"`
    Features    map[string]bool   `json:"features"`
    Limits      map[string]int    `json:"limits"`
}

type DatabaseConfig struct {
    Host     string `json:"host"`
    Port     int    `json:"port"`
    Name     string `json:"name"`
    User     string `json:"user"`
    Password string `json:"password"`
}

// Clone はConfigのディープコピーを返す
func (c *Config) Clone() *Config {
    // マップのディープコピー
    features := make(map[string]bool)
    for k, v := range c.Features {
        features[k] = v
    }

    limits := make(map[string]int)
    for k, v := range c.Limits {
        limits[k] = v
    }

    return &Config{
        AppName:     c.AppName,
        Environment: c.Environment,
        Database: DatabaseConfig{
            Host:     c.Database.Host,
            Port:     c.Database.Port,
            Name:     c.Database.Name,
            User:     c.Database.User,
            Password: c.Database.Password,
        },
        Features: features,
        Limits:   limits,
    }
}

// ConfigTemplates はプリセット設定のテンプレート
type ConfigTemplates struct {
    templates map[string]*Config
}

func NewConfigTemplates() *ConfigTemplates {
    ct := &ConfigTemplates{
        templates: make(map[string]*Config),
    }

    // 開発環境テンプレート
    ct.templates["development"] = &Config{
        AppName:     "MyApp",
        Environment: "development",
        Database: DatabaseConfig{
            Host: "localhost",
            Port: 5432,
            Name: "myapp_dev",
        },
        Features: map[string]bool{
            "debug_mode":   true,
            "hot_reload":   true,
            "verbose_logs": true,
        },
        Limits: map[string]int{
            "max_connections": 10,
            "timeout_seconds": 30,
        },
    }

    // 本番環境テンプレート
    ct.templates["production"] = &Config{
        AppName:     "MyApp",
        Environment: "production",
        Database: DatabaseConfig{
            Host: "db.example.com",
            Port: 5432,
            Name: "myapp_prod",
        },
        Features: map[string]bool{
            "debug_mode":   false,
            "hot_reload":   false,
            "verbose_logs": false,
        },
        Limits: map[string]int{
            "max_connections": 100,
            "timeout_seconds": 10,
        },
    }

    return ct
}

func (ct *ConfigTemplates) CreateConfig(templateName string) (*Config, error) {
    if template, ok := ct.templates[templateName]; ok {
        return template.Clone(), nil
    }
    return nil, fmt.Errorf("template not found: %s", templateName)
}

func main() {
    templates := NewConfigTemplates()

    // 本番環境設定をテンプレートから作成
    prodConfig, _ := templates.CreateConfig("production")

    // 必要に応じてカスタマイズ
    prodConfig.Database.User = "admin"
    prodConfig.Database.Password = "secret123"
    prodConfig.Limits["max_connections"] = 200

    // JSON出力
    jsonData, _ := json.MarshalIndent(prodConfig, "", "  ")
    fmt.Println(string(jsonData))
}
```

## 実践的な例：ゲームオブジェクトのスポーン

```go
package main

import (
    "fmt"
    "math/rand"
)

// Enemy はゲーム内の敵を表す
type Enemy struct {
    Name      string
    HP        int
    MaxHP     int
    Attack    int
    Defense   int
    X, Y      float64
    Abilities []string
}

// Clone はEnemyのディープコピーを返す
func (e *Enemy) Clone() *Enemy {
    abilities := make([]string, len(e.Abilities))
    copy(abilities, e.Abilities)

    return &Enemy{
        Name:      e.Name,
        HP:        e.MaxHP, // 新しい敵はHP満タン
        MaxHP:     e.MaxHP,
        Attack:    e.Attack,
        Defense:   e.Defense,
        Abilities: abilities,
    }
}

func (e *Enemy) SetPosition(x, y float64) {
    e.X = x
    e.Y = y
}

func (e *Enemy) String() string {
    return fmt.Sprintf("%s [HP:%d/%d ATK:%d DEF:%d] at (%.1f, %.1f)",
        e.Name, e.HP, e.MaxHP, e.Attack, e.Defense, e.X, e.Y)
}

// EnemySpawner は敵をスポーンするスポナー
type EnemySpawner struct {
    prototypes map[string]*Enemy
}

func NewEnemySpawner() *EnemySpawner {
    spawner := &EnemySpawner{
        prototypes: make(map[string]*Enemy),
    }

    // 敵のプロトタイプを登録
    spawner.prototypes["goblin"] = &Enemy{
        Name:      "ゴブリン",
        MaxHP:     50,
        Attack:    10,
        Defense:   5,
        Abilities: []string{"こうげき", "にげる"},
    }

    spawner.prototypes["orc"] = &Enemy{
        Name:      "オーク",
        MaxHP:     100,
        Attack:    20,
        Defense:   15,
        Abilities: []string{"こうげき", "パワースマッシュ"},
    }

    spawner.prototypes["dragon"] = &Enemy{
        Name:      "ドラゴン",
        MaxHP:     500,
        Attack:    50,
        Defense:   30,
        Abilities: []string{"こうげき", "ファイアブレス", "つばさでうつ"},
    }

    return spawner
}

func (s *EnemySpawner) Spawn(enemyType string, x, y float64) *Enemy {
    if proto, ok := s.prototypes[enemyType]; ok {
        enemy := proto.Clone()
        enemy.SetPosition(x, y)
        return enemy
    }
    return nil
}

func (s *EnemySpawner) SpawnRandom(x, y float64) *Enemy {
    types := []string{"goblin", "orc"}
    randomType := types[rand.Intn(len(types))]
    return s.Spawn(randomType, x, y)
}

func main() {
    spawner := NewEnemySpawner()

    // 敵をスポーン
    enemies := []*Enemy{
        spawner.Spawn("goblin", 10.0, 20.0),
        spawner.Spawn("goblin", 15.0, 25.0),
        spawner.Spawn("orc", 30.0, 40.0),
        spawner.Spawn("dragon", 100.0, 100.0),
    }

    // 敵一覧を表示
    fmt.Println("スポーンした敵:")
    for _, enemy := range enemies {
        fmt.Println(" ", enemy)
    }

    // ランダムスポーン
    fmt.Println("\nランダムスポーン:")
    for i := 0; i < 3; i++ {
        enemy := spawner.SpawnRandom(float64(i*10), float64(i*10))
        fmt.Println(" ", enemy)
    }
}
```

## Goでのポイント

### JSONを使ったディープコピー

複雑な構造のディープコピーには、JSONシリアライズ/デシリアライズを使う方法があります：

```go
import "encoding/json"

func DeepCopy[T any](src T) (T, error) {
    var dst T
    data, err := json.Marshal(src)
    if err != nil {
        return dst, err
    }
    err = json.Unmarshal(data, &dst)
    return dst, err
}
```

ただし、この方法は：
- パフォーマンスが低い
- unexported fieldはコピーされない
- 関数や channel はコピーできない

### Genericsを活用したCloneable

Go 1.18以降では、ジェネリクスを使えます：

```go
type Cloneable[T any] interface {
    Clone() T
}

func CloneAll[T Cloneable[T]](items []T) []T {
    result := make([]T, len(items))
    for i, item := range items {
        result[i] = item.Clone()
    }
    return result
}
```

## まとめ

- Prototypeは、既存オブジェクトをコピーして新規作成するパターン
- シャローコピーとディープコピーの違いを意識する必要がある
- プロトタイプレジストリを使って、テンプレートから効率的にオブジェクトを生成できる

これで生成に関するパターンの解説は終わりです。次章からは、構造に関するパターンを学びます。
