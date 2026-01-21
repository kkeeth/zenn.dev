---
title: "Abstract Factory パターン"
---

# Abstract Factory パターン

## 概要

Abstract Factoryパターンは、関連するオブジェクト群を、具体的なクラスを指定せずに生成するためのパターンです。「ファクトリのファクトリ」とも呼ばれます。

## 問題

以下のような状況で使用を検討します：

- 関連するオブジェクト群を一貫性を持って生成したい
- システムを複数の「ファミリー」に対応させたい（例：WindowsとmacOS）
- 製品の組み合わせに制約がある（例：WindowsボタンとmacOSチェックボックスを混在させたくない）

## Factory Methodとの違い

| 観点 | Factory Method | Abstract Factory |
|-----|----------------|------------------|
| 対象 | 単一の製品 | 製品のファミリー |
| 継承/実装 | 1つのメソッドをオーバーライド | 複数のファクトリメソッドを持つ |
| 用途 | 1種類のオブジェクト生成 | 関連オブジェクト群の生成 |

## 構造

```
┌─────────────────────┐
│   AbstractFactory   │
│    (interface)      │
├─────────────────────┤
│ + CreateProductA()  │
│ + CreateProductB()  │
└─────────────────────┘
          △
          │
    ┌─────┴─────┐
    │           │
┌───────┐   ┌───────┐
│Factory│   │Factory│
│   1   │   │   2   │
└───────┘   └───────┘
    │           │
    ▼           ▼
ProductA1   ProductA2
ProductB1   ProductB2
```

## 実装

### UIツールキットの例

```go
package main

import "fmt"

// ===== 抽象製品 =====

// Button はボタンを表すインターフェース
type Button interface {
    Render()
    OnClick(func())
}

// Checkbox はチェックボックスを表すインターフェース
type Checkbox interface {
    Render()
    Check()
    Uncheck()
}

// ===== 具象製品（Windows）=====

type WindowsButton struct {
    onClick func()
}

func (b *WindowsButton) Render() {
    fmt.Println("[Windows] ボタンを描画")
}

func (b *WindowsButton) OnClick(handler func()) {
    b.onClick = handler
}

type WindowsCheckbox struct {
    checked bool
}

func (c *WindowsCheckbox) Render() {
    state := "未チェック"
    if c.checked {
        state = "チェック済み"
    }
    fmt.Printf("[Windows] チェックボックスを描画: %s\n", state)
}

func (c *WindowsCheckbox) Check() {
    c.checked = true
}

func (c *WindowsCheckbox) Uncheck() {
    c.checked = false
}

// ===== 具象製品（macOS）=====

type MacButton struct {
    onClick func()
}

func (b *MacButton) Render() {
    fmt.Println("[macOS] ボタンを描画")
}

func (b *MacButton) OnClick(handler func()) {
    b.onClick = handler
}

type MacCheckbox struct {
    checked bool
}

func (c *MacCheckbox) Render() {
    state := "未チェック"
    if c.checked {
        state = "チェック済み"
    }
    fmt.Printf("[macOS] チェックボックスを描画: %s\n", state)
}

func (c *MacCheckbox) Check() {
    c.checked = true
}

func (c *MacCheckbox) Uncheck() {
    c.checked = false
}

// ===== 抽象ファクトリ =====

type GUIFactory interface {
    CreateButton() Button
    CreateCheckbox() Checkbox
}

// ===== 具象ファクトリ =====

type WindowsFactory struct{}

func (f *WindowsFactory) CreateButton() Button {
    return &WindowsButton{}
}

func (f *WindowsFactory) CreateCheckbox() Checkbox {
    return &WindowsCheckbox{}
}

type MacFactory struct{}

func (f *MacFactory) CreateButton() Button {
    return &MacButton{}
}

func (f *MacFactory) CreateCheckbox() Checkbox {
    return &MacCheckbox{}
}

// ===== クライアントコード =====

type Application struct {
    factory  GUIFactory
    button   Button
    checkbox Checkbox
}

func NewApplication(factory GUIFactory) *Application {
    return &Application{
        factory: factory,
    }
}

func (a *Application) CreateUI() {
    a.button = a.factory.CreateButton()
    a.checkbox = a.factory.CreateCheckbox()
}

func (a *Application) Render() {
    a.button.Render()
    a.checkbox.Render()
}

// ファクトリを取得する関数
func GetFactory(osType string) GUIFactory {
    switch osType {
    case "windows":
        return &WindowsFactory{}
    case "mac":
        return &MacFactory{}
    default:
        return &WindowsFactory{}
    }
}

func main() {
    // OS設定に基づいてファクトリを選択
    osType := "mac"
    factory := GetFactory(osType)

    app := NewApplication(factory)
    app.CreateUI()
    app.Render()
}
```

### データベースアクセス層の例

```go
package main

import "fmt"

// ===== 抽象製品 =====

type Connection interface {
    Connect() error
    Close() error
}

type Command interface {
    Execute(query string) error
}

type Transaction interface {
    Begin() error
    Commit() error
    Rollback() error
}

// ===== MySQL実装 =====

type MySQLConnection struct{}

func (c *MySQLConnection) Connect() error {
    fmt.Println("MySQL: 接続を確立")
    return nil
}

func (c *MySQLConnection) Close() error {
    fmt.Println("MySQL: 接続を閉じる")
    return nil
}

type MySQLCommand struct{}

func (c *MySQLCommand) Execute(query string) error {
    fmt.Printf("MySQL: クエリ実行 - %s\n", query)
    return nil
}

type MySQLTransaction struct{}

func (t *MySQLTransaction) Begin() error {
    fmt.Println("MySQL: トランザクション開始")
    return nil
}

func (t *MySQLTransaction) Commit() error {
    fmt.Println("MySQL: コミット")
    return nil
}

func (t *MySQLTransaction) Rollback() error {
    fmt.Println("MySQL: ロールバック")
    return nil
}

// ===== PostgreSQL実装 =====

type PostgreSQLConnection struct{}

func (c *PostgreSQLConnection) Connect() error {
    fmt.Println("PostgreSQL: 接続を確立")
    return nil
}

func (c *PostgreSQLConnection) Close() error {
    fmt.Println("PostgreSQL: 接続を閉じる")
    return nil
}

type PostgreSQLCommand struct{}

func (c *PostgreSQLCommand) Execute(query string) error {
    fmt.Printf("PostgreSQL: クエリ実行 - %s\n", query)
    return nil
}

type PostgreSQLTransaction struct{}

func (t *PostgreSQLTransaction) Begin() error {
    fmt.Println("PostgreSQL: トランザクション開始")
    return nil
}

func (t *PostgreSQLTransaction) Commit() error {
    fmt.Println("PostgreSQL: コミット")
    return nil
}

func (t *PostgreSQLTransaction) Rollback() error {
    fmt.Println("PostgreSQL: ロールバック")
    return nil
}

// ===== 抽象ファクトリ =====

type DatabaseFactory interface {
    CreateConnection() Connection
    CreateCommand() Command
    CreateTransaction() Transaction
}

// ===== 具象ファクトリ =====

type MySQLFactory struct{}

func (f *MySQLFactory) CreateConnection() Connection {
    return &MySQLConnection{}
}

func (f *MySQLFactory) CreateCommand() Command {
    return &MySQLCommand{}
}

func (f *MySQLFactory) CreateTransaction() Transaction {
    return &MySQLTransaction{}
}

type PostgreSQLFactory struct{}

func (f *PostgreSQLFactory) CreateConnection() Connection {
    return &PostgreSQLConnection{}
}

func (f *PostgreSQLFactory) CreateCommand() Command {
    return &PostgreSQLCommand{}
}

func (f *PostgreSQLFactory) CreateTransaction() Transaction {
    return &PostgreSQLTransaction{}
}

// ===== リポジトリ =====

type UserRepository struct {
    conn        Connection
    command     Command
    transaction Transaction
}

func NewUserRepository(factory DatabaseFactory) *UserRepository {
    return &UserRepository{
        conn:        factory.CreateConnection(),
        command:     factory.CreateCommand(),
        transaction: factory.CreateTransaction(),
    }
}

func (r *UserRepository) Save(name string) error {
    r.conn.Connect()
    defer r.conn.Close()

    r.transaction.Begin()
    err := r.command.Execute(fmt.Sprintf("INSERT INTO users (name) VALUES ('%s')", name))
    if err != nil {
        r.transaction.Rollback()
        return err
    }
    r.transaction.Commit()
    return nil
}

func main() {
    // 設定に応じてファクトリを選択
    var factory DatabaseFactory = &PostgreSQLFactory{}

    repo := NewUserRepository(factory)
    repo.Save("田中太郎")
}
```

## 実践的な例：クラウドプロバイダー

```go
package main

import "fmt"

// ===== 抽象製品 =====

type VirtualMachine interface {
    Start()
    Stop()
    GetStatus() string
}

type Storage interface {
    Upload(data []byte) error
    Download(key string) ([]byte, error)
}

type Network interface {
    CreateVPC(cidr string) error
    CreateSubnet(vpcID, cidr string) error
}

// ===== AWS実装 =====

type AWSEC2 struct {
    running bool
}

func (e *AWSEC2) Start() {
    fmt.Println("AWS EC2: インスタンスを起動")
    e.running = true
}

func (e *AWSEC2) Stop() {
    fmt.Println("AWS EC2: インスタンスを停止")
    e.running = false
}

func (e *AWSEC2) GetStatus() string {
    if e.running {
        return "running"
    }
    return "stopped"
}

type AWSS3 struct{}

func (s *AWSS3) Upload(data []byte) error {
    fmt.Printf("AWS S3: %d bytes をアップロード\n", len(data))
    return nil
}

func (s *AWSS3) Download(key string) ([]byte, error) {
    fmt.Printf("AWS S3: %s をダウンロード\n", key)
    return []byte("data"), nil
}

type AWSVPC struct{}

func (v *AWSVPC) CreateVPC(cidr string) error {
    fmt.Printf("AWS VPC: VPCを作成 (CIDR: %s)\n", cidr)
    return nil
}

func (v *AWSVPC) CreateSubnet(vpcID, cidr string) error {
    fmt.Printf("AWS VPC: サブネットを作成 (VPC: %s, CIDR: %s)\n", vpcID, cidr)
    return nil
}

// ===== GCP実装 =====

type GCPComputeEngine struct {
    running bool
}

func (c *GCPComputeEngine) Start() {
    fmt.Println("GCP Compute Engine: VMを起動")
    c.running = true
}

func (c *GCPComputeEngine) Stop() {
    fmt.Println("GCP Compute Engine: VMを停止")
    c.running = false
}

func (c *GCPComputeEngine) GetStatus() string {
    if c.running {
        return "RUNNING"
    }
    return "TERMINATED"
}

type GCPCloudStorage struct{}

func (s *GCPCloudStorage) Upload(data []byte) error {
    fmt.Printf("GCP Cloud Storage: %d bytes をアップロード\n", len(data))
    return nil
}

func (s *GCPCloudStorage) Download(key string) ([]byte, error) {
    fmt.Printf("GCP Cloud Storage: %s をダウンロード\n", key)
    return []byte("data"), nil
}

type GCPVPC struct{}

func (v *GCPVPC) CreateVPC(cidr string) error {
    fmt.Printf("GCP VPC: VPCネットワークを作成 (CIDR: %s)\n", cidr)
    return nil
}

func (v *GCPVPC) CreateSubnet(vpcID, cidr string) error {
    fmt.Printf("GCP VPC: サブネットを作成 (Network: %s, CIDR: %s)\n", vpcID, cidr)
    return nil
}

// ===== 抽象ファクトリ =====

type CloudFactory interface {
    CreateVM() VirtualMachine
    CreateStorage() Storage
    CreateNetwork() Network
}

type AWSFactory struct{}

func (f *AWSFactory) CreateVM() VirtualMachine { return &AWSEC2{} }
func (f *AWSFactory) CreateStorage() Storage   { return &AWSS3{} }
func (f *AWSFactory) CreateNetwork() Network   { return &AWSVPC{} }

type GCPFactory struct{}

func (f *GCPFactory) CreateVM() VirtualMachine { return &GCPComputeEngine{} }
func (f *GCPFactory) CreateStorage() Storage   { return &GCPCloudStorage{} }
func (f *GCPFactory) CreateNetwork() Network   { return &GCPVPC{} }

// ===== インフラ管理 =====

type Infrastructure struct {
    vm      VirtualMachine
    storage Storage
    network Network
}

func NewInfrastructure(factory CloudFactory) *Infrastructure {
    return &Infrastructure{
        vm:      factory.CreateVM(),
        storage: factory.CreateStorage(),
        network: factory.CreateNetwork(),
    }
}

func (i *Infrastructure) Setup() {
    i.network.CreateVPC("10.0.0.0/16")
    i.network.CreateSubnet("vpc-1", "10.0.1.0/24")
    i.vm.Start()
    i.storage.Upload([]byte("初期設定データ"))
}

func main() {
    // 環境変数などで切り替え
    provider := "gcp"

    var factory CloudFactory
    switch provider {
    case "aws":
        factory = &AWSFactory{}
    case "gcp":
        factory = &GCPFactory{}
    }

    infra := NewInfrastructure(factory)
    infra.Setup()
}
```

## Goでのポイント

### インターフェースの組み合わせ

Goでは小さなインターフェースを組み合わせることが推奨されます：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// 複数のインターフェースを組み合わせ
type ReadWriter interface {
    Reader
    Writer
}
```

### 関数型のアプローチ

シンプルなケースでは、関数の集合として表現することもできます：

```go
type Factory struct {
    CreateButton   func() Button
    CreateCheckbox func() Checkbox
}

func NewWindowsFactory() *Factory {
    return &Factory{
        CreateButton:   func() Button { return &WindowsButton{} },
        CreateCheckbox: func() Checkbox { return &WindowsCheckbox{} },
    }
}
```

## まとめ

- Abstract Factoryは、関連するオブジェクト群を一貫性を持って生成する
- 製品ファミリー（Windows/macOS、AWS/GCPなど）の切り替えが容易
- Goではインターフェースを活用してシンプルに実装できる

次章では、複雑なオブジェクトを段階的に構築するBuilderパターンを学びます。
