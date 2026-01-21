---
title: "Flyweight パターン"
---

# Flyweight パターン

## 概要

Flyweightパターンは、多数の細粒度オブジェクトを効率的に共有することで、メモリ使用量を削減するパターンです。

## 問題

以下のような状況で使用を検討します：

- 大量の類似オブジェクトを生成する必要がある
- オブジェクトの大部分の状態が共有可能（intrinsic state）
- メモリ使用量を削減したい

## 内部状態と外部状態

- **内部状態（Intrinsic State）**: オブジェクト間で共有できる不変のデータ
- **外部状態（Extrinsic State）**: コンテキストによって異なるデータ（クライアントが管理）

## 構造

```
┌─────────────────────┐
│   FlyweightFactory  │
├─────────────────────┤
│ - flyweights: map   │
├─────────────────────┤
│ + GetFlyweight()    │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│     Flyweight       │
│    (interface)      │
├─────────────────────┤
│ + Operation(extrin) │
└─────────────────────┘
```

## 実装

### 文字レンダリングの例

```go
package main

import "fmt"

// CharacterFlyweight は文字のFlyweight
type CharacterFlyweight struct {
    // 内部状態（共有される）
    char     rune
    fontName string
    fontSize int
}

// Render は外部状態（位置）を受け取って描画
func (c *CharacterFlyweight) Render(x, y int) {
    fmt.Printf("文字 '%c' を (%d, %d) に描画 [フォント: %s, サイズ: %d]\n",
        c.char, x, y, c.fontName, c.fontSize)
}

// CharacterFactory はFlyweightを管理するファクトリ
type CharacterFactory struct {
    characters map[string]*CharacterFlyweight
}

func NewCharacterFactory() *CharacterFactory {
    return &CharacterFactory{
        characters: make(map[string]*CharacterFlyweight),
    }
}

func (f *CharacterFactory) GetCharacter(char rune, fontName string, fontSize int) *CharacterFlyweight {
    key := fmt.Sprintf("%c-%s-%d", char, fontName, fontSize)

    if flyweight, ok := f.characters[key]; ok {
        fmt.Printf("  [Cache Hit] '%c'\n", char)
        return flyweight
    }

    fmt.Printf("  [Creating] '%c'\n", char)
    flyweight := &CharacterFlyweight{
        char:     char,
        fontName: fontName,
        fontSize: fontSize,
    }
    f.characters[key] = flyweight
    return flyweight
}

func (f *CharacterFactory) GetCount() int {
    return len(f.characters)
}

func main() {
    factory := NewCharacterFactory()

    // テキストをレンダリング（同じ文字は共有される）
    text := "HELLO WORLD"
    x := 0

    fmt.Println("=== 文字オブジェクトの取得 ===")
    for _, char := range text {
        if char == ' ' {
            x += 10
            continue
        }
        flyweight := factory.GetCharacter(char, "Arial", 12)
        flyweight.Render(x, 0)
        x += 15
    }

    fmt.Printf("\n文字数: %d, 作成されたオブジェクト数: %d\n",
        len(text)-1, factory.GetCount()) // スペースを除く
}
```

### ゲームの木オブジェクトの例

```go
package main

import (
    "fmt"
    "math/rand"
)

// TreeType は木の種類（内部状態 - 共有される）
type TreeType struct {
    name    string
    color   string
    texture string // 大きなテクスチャデータを想定
}

func (t *TreeType) Draw(x, y int) {
    fmt.Printf("  [%s] (%d, %d) に描画 - 色: %s\n", t.name, x, y, t.color)
}

// TreeFactory はTreeTypeを管理するファクトリ
type TreeFactory struct {
    treeTypes map[string]*TreeType
}

func NewTreeFactory() *TreeFactory {
    return &TreeFactory{
        treeTypes: make(map[string]*TreeType),
    }
}

func (f *TreeFactory) GetTreeType(name, color, texture string) *TreeType {
    key := name + "-" + color

    if treeType, ok := f.treeTypes[key]; ok {
        return treeType
    }

    treeType := &TreeType{
        name:    name,
        color:   color,
        texture: texture,
    }
    f.treeTypes[key] = treeType
    fmt.Printf("[Factory] 新しい木タイプを作成: %s (%s)\n", name, color)
    return treeType
}

// Tree は木のインスタンス（外部状態を持つ）
type Tree struct {
    x, y     int       // 外部状態（位置）
    treeType *TreeType // 内部状態への参照（共有）
}

func NewTree(x, y int, treeType *TreeType) *Tree {
    return &Tree{x: x, y: y, treeType: treeType}
}

func (t *Tree) Draw() {
    t.treeType.Draw(t.x, t.y)
}

// Forest は森（多数の木を管理）
type Forest struct {
    trees   []*Tree
    factory *TreeFactory
}

func NewForest() *Forest {
    return &Forest{
        trees:   []*Tree{},
        factory: NewTreeFactory(),
    }
}

func (f *Forest) PlantTree(x, y int, name, color, texture string) {
    treeType := f.factory.GetTreeType(name, color, texture)
    tree := NewTree(x, y, treeType)
    f.trees = append(f.trees, tree)
}

func (f *Forest) Draw() {
    fmt.Println("\n=== 森を描画 ===")
    for _, tree := range f.trees {
        tree.Draw()
    }
}

func (f *Forest) GetStats() {
    fmt.Printf("\n統計: 木の数=%d, 木タイプの数=%d\n",
        len(f.trees), len(f.factory.treeTypes))
}

func main() {
    forest := NewForest()

    // 1000本の木を植える（実際には3種類のタイプのみ）
    treeData := []struct {
        name, color, texture string
    }{
        {"オーク", "緑", "oak_texture.png"},
        {"松", "深緑", "pine_texture.png"},
        {"白樺", "黄緑", "birch_texture.png"},
    }

    fmt.Println("=== 木を植える ===")
    for i := 0; i < 10; i++ {
        data := treeData[rand.Intn(len(treeData))]
        forest.PlantTree(rand.Intn(500), rand.Intn(500), data.name, data.color, data.texture)
    }

    forest.Draw()
    forest.GetStats()
}
```

## 実践的な例：アイコンキャッシュ

```go
package main

import (
    "fmt"
    "sync"
)

// Icon はアイコンの内部状態
type Icon struct {
    name   string
    path   string
    width  int
    height int
    // data []byte // 実際にはバイナリデータ
}

func (i *Icon) GetSize() string {
    return fmt.Sprintf("%dx%d", i.width, i.height)
}

// IconCache はアイコンのFlyweightファクトリ（スレッドセーフ）
type IconCache struct {
    mu    sync.RWMutex
    icons map[string]*Icon
}

var (
    iconCache *IconCache
    once      sync.Once
)

func GetIconCache() *IconCache {
    once.Do(func() {
        iconCache = &IconCache{
            icons: make(map[string]*Icon),
        }
    })
    return iconCache
}

func (c *IconCache) GetIcon(name string) *Icon {
    c.mu.RLock()
    if icon, ok := c.icons[name]; ok {
        c.mu.RUnlock()
        fmt.Printf("[Cache Hit] %s\n", name)
        return icon
    }
    c.mu.RUnlock()

    // キャッシュミス - 新規作成
    c.mu.Lock()
    defer c.mu.Unlock()

    // ダブルチェック
    if icon, ok := c.icons[name]; ok {
        return icon
    }

    fmt.Printf("[Loading] %s\n", name)
    icon := loadIcon(name) // 実際にはファイルから読み込む
    c.icons[name] = icon
    return icon
}

func loadIcon(name string) *Icon {
    // 実際にはファイルシステムから読み込む
    return &Icon{
        name:   name,
        path:   fmt.Sprintf("/icons/%s.png", name),
        width:  32,
        height: 32,
    }
}

// Button はボタン（外部状態を持つ）
type Button struct {
    x, y  int
    label string
    icon  *Icon // 共有されるアイコン
}

func NewButton(x, y int, label, iconName string) *Button {
    cache := GetIconCache()
    return &Button{
        x:     x,
        y:     y,
        label: label,
        icon:  cache.GetIcon(iconName),
    }
}

func (b *Button) Render() {
    fmt.Printf("Button[%s] at (%d,%d) with icon %s (%s)\n",
        b.label, b.x, b.y, b.icon.name, b.icon.GetSize())
}

func main() {
    fmt.Println("=== ボタンを作成 ===")

    buttons := []*Button{
        NewButton(10, 10, "保存", "save"),
        NewButton(50, 10, "開く", "open"),
        NewButton(90, 10, "保存", "save"),  // 同じアイコン
        NewButton(130, 10, "閉じる", "close"),
        NewButton(170, 10, "開く", "open"),  // 同じアイコン
        NewButton(210, 10, "保存", "save"),  // 同じアイコン
    }

    fmt.Println("\n=== ボタンを描画 ===")
    for _, btn := range buttons {
        btn.Render()
    }

    cache := GetIconCache()
    fmt.Printf("\nボタン数: %d, キャッシュされたアイコン数: %d\n",
        len(buttons), len(cache.icons))
}
```

## 実践的な例：接続プール

```go
package main

import (
    "fmt"
    "sync"
)

// Connection はDB接続（重いリソース）
type Connection struct {
    id     int
    inUse  bool
    host   string
}

func (c *Connection) Query(sql string) {
    fmt.Printf("[Conn-%d] Executing: %s\n", c.id, sql)
}

// ConnectionPool は接続プール
type ConnectionPool struct {
    mu          sync.Mutex
    connections []*Connection
    maxSize     int
    host        string
    nextID      int
}

func NewConnectionPool(host string, maxSize int) *ConnectionPool {
    return &ConnectionPool{
        connections: make([]*Connection, 0),
        maxSize:     maxSize,
        host:        host,
        nextID:      1,
    }
}

func (p *ConnectionPool) Acquire() (*Connection, error) {
    p.mu.Lock()
    defer p.mu.Unlock()

    // 空いている接続を探す
    for _, conn := range p.connections {
        if !conn.inUse {
            conn.inUse = true
            fmt.Printf("[Pool] 既存の接続を再利用: Conn-%d\n", conn.id)
            return conn, nil
        }
    }

    // 新規作成が可能な場合
    if len(p.connections) < p.maxSize {
        conn := &Connection{
            id:    p.nextID,
            inUse: true,
            host:  p.host,
        }
        p.nextID++
        p.connections = append(p.connections, conn)
        fmt.Printf("[Pool] 新規接続を作成: Conn-%d\n", conn.id)
        return conn, nil
    }

    return nil, fmt.Errorf("接続プールが満杯です")
}

func (p *ConnectionPool) Release(conn *Connection) {
    p.mu.Lock()
    defer p.mu.Unlock()

    conn.inUse = false
    fmt.Printf("[Pool] 接続を解放: Conn-%d\n", conn.id)
}

func (p *ConnectionPool) Stats() {
    inUse := 0
    for _, conn := range p.connections {
        if conn.inUse {
            inUse++
        }
    }
    fmt.Printf("[Pool] 総数: %d, 使用中: %d, 空き: %d\n",
        len(p.connections), inUse, len(p.connections)-inUse)
}

func main() {
    pool := NewConnectionPool("localhost:5432", 3)

    fmt.Println("=== 接続を取得 ===")
    conn1, _ := pool.Acquire()
    conn1.Query("SELECT * FROM users")

    conn2, _ := pool.Acquire()
    conn2.Query("SELECT * FROM products")

    pool.Stats()

    fmt.Println("\n=== 接続を解放 ===")
    pool.Release(conn1)
    pool.Stats()

    fmt.Println("\n=== 再度接続を取得（再利用される）===")
    conn3, _ := pool.Acquire()
    conn3.Query("SELECT * FROM orders")
    pool.Stats()
}
```

## Goでのポイント

### sync.Poolの活用

Goの標準ライブラリには`sync.Pool`があり、Flyweightの一形態として使えます：

```go
var bufferPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func ProcessData(data []byte) {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()

    buf.Write(data)
    // 処理...
}
```

### インターンパターン

文字列のインターンもFlyweightの一例：

```go
var stringCache = make(map[string]string)
var mu sync.RWMutex

func Intern(s string) string {
    mu.RLock()
    if cached, ok := stringCache[s]; ok {
        mu.RUnlock()
        return cached
    }
    mu.RUnlock()

    mu.Lock()
    defer mu.Unlock()
    stringCache[s] = s
    return s
}
```

## まとめ

- Flyweightは大量の類似オブジェクトを効率的に共有する
- 内部状態（共有可能）と外部状態（クライアント管理）を分離
- Goでは`sync.Pool`や独自のキャッシュで実装

次章では、オブジェクトへのアクセスを制御するProxyパターンを学びます。
