---
title: "Adapter パターン"
---

# Adapter パターン

## 概要

Adapterパターンは、互換性のないインターフェース同士をつなぐパターンです。既存のクラスを修正せずに、異なるインターフェースを持つクラスと協調して動作させることができます。

## 問題

以下のような状況で使用を検討します：

- サードパーティのライブラリを既存のコードに統合したい
- レガシーコードを新しいシステムに組み込みたい
- 異なるインターフェースを持つオブジェクトを統一的に扱いたい

## 構造

```
┌─────────────┐       ┌─────────────┐
│   Client    │──────▶│   Target    │
└─────────────┘       │ (interface) │
                      └─────────────┘
                            △
                            │
                      ┌─────────────┐
                      │   Adapter   │
                      ├─────────────┤
                      │ - adaptee   │───▶ ┌─────────────┐
                      └─────────────┘     │   Adaptee   │
                                          │  (既存実装)  │
                                          └─────────────┘
```

## 実装

### オブジェクトアダプタ（コンポジション）

```go
package main

import "fmt"

// ===== 既存のインターフェース（Target）=====

// MediaPlayer は音楽プレイヤーのインターフェース
type MediaPlayer interface {
    Play(filename string)
}

// ===== 既存の実装（Adaptee）=====

// AdvancedMediaPlayer は高度なメディアプレイヤー
type AdvancedMediaPlayer interface {
    PlayVLC(filename string)
    PlayMP4(filename string)
}

// VLCPlayer はVLC形式を再生する
type VLCPlayer struct{}

func (v *VLCPlayer) PlayVLC(filename string) {
    fmt.Printf("VLCプレイヤーで再生中: %s\n", filename)
}

func (v *VLCPlayer) PlayMP4(filename string) {
    // VLCPlayerはMP4をサポートしない
}

// MP4Player はMP4形式を再生する
type MP4Player struct{}

func (m *MP4Player) PlayVLC(filename string) {
    // MP4PlayerはVLCをサポートしない
}

func (m *MP4Player) PlayMP4(filename string) {
    fmt.Printf("MP4プレイヤーで再生中: %s\n", filename)
}

// ===== アダプタ =====

// MediaAdapter は AdvancedMediaPlayer を MediaPlayer に適合させる
type MediaAdapter struct {
    advancedPlayer AdvancedMediaPlayer
}

func NewMediaAdapter(audioType string) *MediaAdapter {
    adapter := &MediaAdapter{}
    switch audioType {
    case "vlc":
        adapter.advancedPlayer = &VLCPlayer{}
    case "mp4":
        adapter.advancedPlayer = &MP4Player{}
    }
    return adapter
}

func (a *MediaAdapter) Play(filename string) {
    if vlcPlayer, ok := a.advancedPlayer.(*VLCPlayer); ok {
        vlcPlayer.PlayVLC(filename)
    } else if mp4Player, ok := a.advancedPlayer.(*MP4Player); ok {
        mp4Player.PlayMP4(filename)
    }
}

// ===== クライアント =====

// AudioPlayer は統一されたメディアプレイヤー
type AudioPlayer struct {
    mediaAdapter *MediaAdapter
}

func (p *AudioPlayer) Play(audioType, filename string) {
    switch audioType {
    case "mp3":
        fmt.Printf("MP3を再生中: %s\n", filename)
    case "vlc", "mp4":
        p.mediaAdapter = NewMediaAdapter(audioType)
        p.mediaAdapter.Play(filename)
    default:
        fmt.Printf("サポートされていない形式: %s\n", audioType)
    }
}

func main() {
    player := &AudioPlayer{}

    player.Play("mp3", "song.mp3")
    player.Play("mp4", "movie.mp4")
    player.Play("vlc", "video.vlc")
    player.Play("avi", "movie.avi")
}
```

### 埋め込みを使った実装

Goの埋め込み機能を使うと、よりシンプルに実装できます：

```go
package main

import "fmt"

// ===== 期待するインターフェース =====

// Printer は印刷機能を提供する
type Printer interface {
    Print(msg string)
}

// ===== 既存の実装（Adaptee）=====

// LegacyPrinter は古い形式のプリンター
type LegacyPrinter struct{}

func (l *LegacyPrinter) PrintLegacy(s string) {
    fmt.Printf("[Legacy] %s\n", s)
}

// ===== アダプタ（埋め込みを使用）=====

// LegacyPrinterAdapter は LegacyPrinter を Printer に適合させる
type LegacyPrinterAdapter struct {
    *LegacyPrinter // 埋め込み
}

// Print は Printer インターフェースを実装
func (a *LegacyPrinterAdapter) Print(msg string) {
    a.PrintLegacy(msg) // 埋め込まれたメソッドを呼び出す
}

func main() {
    // LegacyPrinter を Printer として使用
    var printer Printer = &LegacyPrinterAdapter{
        LegacyPrinter: &LegacyPrinter{},
    }

    printer.Print("Hello, Adapter Pattern!")
}
```

## 実践的な例：外部APIの統合

```go
package main

import (
    "encoding/json"
    "fmt"
)

// ===== 自社システムのインターフェース =====

type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

type UserProvider interface {
    GetUser(id string) (*User, error)
    ListUsers() ([]*User, error)
}

// ===== 外部サービスA（既存の実装）=====

type ExternalServiceA struct{}

type ExternalUserA struct {
    UserID      string `json:"user_id"`
    DisplayName string `json:"display_name"`
    EmailAddr   string `json:"email_addr"`
}

func (s *ExternalServiceA) FetchUser(userID string) (*ExternalUserA, error) {
    // 実際にはAPIコールを行う
    return &ExternalUserA{
        UserID:      userID,
        DisplayName: "サービスA ユーザー",
        EmailAddr:   "user@service-a.com",
    }, nil
}

func (s *ExternalServiceA) FetchAllUsers() ([]*ExternalUserA, error) {
    return []*ExternalUserA{
        {UserID: "a1", DisplayName: "ユーザー1", EmailAddr: "user1@a.com"},
        {UserID: "a2", DisplayName: "ユーザー2", EmailAddr: "user2@a.com"},
    }, nil
}

// ===== 外部サービスB（既存の実装）=====

type ExternalServiceB struct{}

type ExternalUserB struct {
    ID       int    `json:"id"`
    FullName string `json:"full_name"`
    Contact  string `json:"contact"`
}

func (s *ExternalServiceB) GetUserByID(id int) (*ExternalUserB, error) {
    return &ExternalUserB{
        ID:       id,
        FullName: "サービスB ユーザー",
        Contact:  "user@service-b.com",
    }, nil
}

func (s *ExternalServiceB) GetUsers() ([]*ExternalUserB, error) {
    return []*ExternalUserB{
        {ID: 1, FullName: "Bユーザー1", Contact: "b1@b.com"},
        {ID: 2, FullName: "Bユーザー2", Contact: "b2@b.com"},
    }, nil
}

// ===== アダプタA =====

type ServiceAAdapter struct {
    service *ExternalServiceA
}

func NewServiceAAdapter() *ServiceAAdapter {
    return &ServiceAAdapter{service: &ExternalServiceA{}}
}

func (a *ServiceAAdapter) GetUser(id string) (*User, error) {
    extUser, err := a.service.FetchUser(id)
    if err != nil {
        return nil, err
    }
    return &User{
        ID:    extUser.UserID,
        Name:  extUser.DisplayName,
        Email: extUser.EmailAddr,
    }, nil
}

func (a *ServiceAAdapter) ListUsers() ([]*User, error) {
    extUsers, err := a.service.FetchAllUsers()
    if err != nil {
        return nil, err
    }
    users := make([]*User, len(extUsers))
    for i, eu := range extUsers {
        users[i] = &User{
            ID:    eu.UserID,
            Name:  eu.DisplayName,
            Email: eu.EmailAddr,
        }
    }
    return users, nil
}

// ===== アダプタB =====

type ServiceBAdapter struct {
    service *ExternalServiceB
}

func NewServiceBAdapter() *ServiceBAdapter {
    return &ServiceBAdapter{service: &ExternalServiceB{}}
}

func (a *ServiceBAdapter) GetUser(id string) (*User, error) {
    // 文字列IDを整数に変換（簡略化のため）
    var intID int
    fmt.Sscanf(id, "%d", &intID)

    extUser, err := a.service.GetUserByID(intID)
    if err != nil {
        return nil, err
    }
    return &User{
        ID:    fmt.Sprintf("%d", extUser.ID),
        Name:  extUser.FullName,
        Email: extUser.Contact,
    }, nil
}

func (a *ServiceBAdapter) ListUsers() ([]*User, error) {
    extUsers, err := a.service.GetUsers()
    if err != nil {
        return nil, err
    }
    users := make([]*User, len(extUsers))
    for i, eu := range extUsers {
        users[i] = &User{
            ID:    fmt.Sprintf("%d", eu.ID),
            Name:  eu.FullName,
            Email: eu.Contact,
        }
    }
    return users, nil
}

// ===== クライアントコード =====

type UserService struct {
    provider UserProvider
}

func NewUserService(provider UserProvider) *UserService {
    return &UserService{provider: provider}
}

func (s *UserService) DisplayUsers() {
    users, _ := s.provider.ListUsers()
    for _, u := range users {
        data, _ := json.Marshal(u)
        fmt.Println(string(data))
    }
}

func main() {
    fmt.Println("=== Service A ===")
    serviceA := NewUserService(NewServiceAAdapter())
    serviceA.DisplayUsers()

    fmt.Println("\n=== Service B ===")
    serviceB := NewUserService(NewServiceBAdapter())
    serviceB.DisplayUsers()
}
```

## 実践的な例：ロガーの統合

```go
package main

import (
    "fmt"
    "log"
    "os"
)

// ===== アプリケーションのロガーインターフェース =====

type Logger interface {
    Debug(msg string)
    Info(msg string)
    Error(msg string)
}

// ===== 標準ライブラリのlog（Adaptee）=====

// StdLogAdapter は標準ライブラリの log を Logger に適合させる
type StdLogAdapter struct {
    logger *log.Logger
}

func NewStdLogAdapter() *StdLogAdapter {
    return &StdLogAdapter{
        logger: log.New(os.Stdout, "", log.LstdFlags),
    }
}

func (a *StdLogAdapter) Debug(msg string) {
    a.logger.Printf("[DEBUG] %s", msg)
}

func (a *StdLogAdapter) Info(msg string) {
    a.logger.Printf("[INFO] %s", msg)
}

func (a *StdLogAdapter) Error(msg string) {
    a.logger.Printf("[ERROR] %s", msg)
}

// ===== カスタムロガー（Adaptee）=====

type CustomLogger struct {
    prefix string
}

func (c *CustomLogger) Log(level, message string) {
    fmt.Printf("%s %s: %s\n", c.prefix, level, message)
}

// CustomLogAdapter は CustomLogger を Logger に適合させる
type CustomLogAdapter struct {
    logger *CustomLogger
}

func NewCustomLogAdapter(prefix string) *CustomLogAdapter {
    return &CustomLogAdapter{
        logger: &CustomLogger{prefix: prefix},
    }
}

func (a *CustomLogAdapter) Debug(msg string) {
    a.logger.Log("DEBUG", msg)
}

func (a *CustomLogAdapter) Info(msg string) {
    a.logger.Log("INFO", msg)
}

func (a *CustomLogAdapter) Error(msg string) {
    a.logger.Log("ERROR", msg)
}

// ===== アプリケーション =====

type Application struct {
    logger Logger
}

func NewApplication(logger Logger) *Application {
    return &Application{logger: logger}
}

func (a *Application) Run() {
    a.logger.Info("アプリケーションを開始します")
    a.logger.Debug("初期化処理中...")
    a.logger.Info("処理完了")
    a.logger.Error("テストエラー")
}

func main() {
    fmt.Println("=== 標準ロガー ===")
    app1 := NewApplication(NewStdLogAdapter())
    app1.Run()

    fmt.Println("\n=== カスタムロガー ===")
    app2 := NewApplication(NewCustomLogAdapter("[MyApp]"))
    app2.Run()
}
```

## Goでのポイント

### インターフェースの活用

Goの暗黙的インターフェース実装はAdapterパターンと相性が良いです：

```go
// io.Readerを期待する関数
func ReadAll(r io.Reader) ([]byte, error) {
    return io.ReadAll(r)
}

// strings.Readerは自動的にio.Readerを実装
reader := strings.NewReader("Hello")
data, _ := ReadAll(reader)
```

### 関数アダプタ

関数型を使った軽量なアダプタも可能です：

```go
// http.HandlerFunc は func(w, r) を http.Handler に適合させる
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello"))
})
```

## Two-Way Adapter

双方向のアダプタを作ることもできます：

```go
type TwoWayAdapter struct {
    serviceA *ServiceA
    serviceB *ServiceB
}

// ServiceAとして振る舞う
func (a *TwoWayAdapter) MethodA() { /* ServiceBを使ってMethodAを実装 */ }

// ServiceBとして振る舞う
func (a *TwoWayAdapter) MethodB() { /* ServiceAを使ってMethodBを実装 */ }
```

## まとめ

- Adapterは、互換性のないインターフェースをつなぐパターン
- Goでは埋め込みとインターフェースを活用してシンプルに実装できる
- 外部ライブラリやレガシーコードの統合に有効

次章では、抽象と実装を分離するBridgeパターンを学びます。
