---
title: "Singleton パターン"
---

# Singleton パターン

## 概要

Singletonパターンは、クラスのインスタンスが1つだけであることを保証し、そのインスタンスへのグローバルなアクセスポイントを提供するパターンです。

## 問題

以下のような状況で使用を検討します：

- アプリケーション全体で共有する設定を管理したい
- データベースへのコネクションプールを1つだけ持ちたい
- ログ出力を一元管理したい

## 構造

```
┌─────────────────────────────────────┐
│           Singleton                  │
├─────────────────────────────────────┤
│ - instance: *Singleton              │
│ - once: sync.Once                   │
├─────────────────────────────────────┤
│ + GetInstance(): *Singleton         │
│ + DoSomething()                     │
└─────────────────────────────────────┘
```

## 実装

### 基本的な実装（sync.Onceを使用）

```go
package singleton

import "sync"

// Config はアプリケーション設定を保持するSingleton
type Config struct {
    DatabaseURL string
    APIKey      string
    Debug       bool
}

var (
    instance *Config
    once     sync.Once
)

// GetInstance は Config のシングルトンインスタンスを返す
func GetInstance() *Config {
    once.Do(func() {
        instance = &Config{
            DatabaseURL: "localhost:5432",
            APIKey:      "default-key",
            Debug:       false,
        }
    })
    return instance
}
```

### 使用例

```go
package main

import (
    "fmt"
    "singleton"
)

func main() {
    // どこから呼んでも同じインスタンスを取得
    config1 := singleton.GetInstance()
    config2 := singleton.GetInstance()

    // 同一インスタンスであることを確認
    fmt.Println(config1 == config2) // true

    // 設定を変更
    config1.Debug = true

    // 別の場所から取得しても変更が反映されている
    fmt.Println(config2.Debug) // true
}
```

### ゴルーチンセーフの確認

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    // 100個のゴルーチンから同時にアクセス
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            config := singleton.GetInstance()
            fmt.Printf("Goroutine %d: %p\n", id, config)
        }(i)
    }

    wg.Wait()
    // すべて同じアドレスが出力される
}
```

### 遅延初期化なしの実装

パッケージの初期化時にインスタンスを作成する方法もあります：

```go
package singleton

// パッケージ初期化時に作成
var instance = &Config{
    DatabaseURL: "localhost:5432",
    APIKey:      "default-key",
    Debug:       false,
}

// GetInstance はインスタンスを返す
func GetInstance() *Config {
    return instance
}
```

## 実践的な例：ロガー

```go
package logger

import (
    "log"
    "os"
    "sync"
)

// Logger はアプリケーション全体で共有されるロガー
type Logger struct {
    logger *log.Logger
    level  int
}

const (
    DEBUG = iota
    INFO
    WARN
    ERROR
)

var (
    instance *Logger
    once     sync.Once
)

// GetLogger はロガーのシングルトンインスタンスを返す
func GetLogger() *Logger {
    once.Do(func() {
        instance = &Logger{
            logger: log.New(os.Stdout, "", log.LstdFlags),
            level:  INFO,
        }
    })
    return instance
}

// SetLevel はログレベルを設定する
func (l *Logger) SetLevel(level int) {
    l.level = level
}

// Debug はデバッグログを出力する
func (l *Logger) Debug(msg string) {
    if l.level <= DEBUG {
        l.logger.Printf("[DEBUG] %s", msg)
    }
}

// Info は情報ログを出力する
func (l *Logger) Info(msg string) {
    if l.level <= INFO {
        l.logger.Printf("[INFO] %s", msg)
    }
}

// Error はエラーログを出力する
func (l *Logger) Error(msg string) {
    if l.level <= ERROR {
        l.logger.Printf("[ERROR] %s", msg)
    }
}
```

使用例：

```go
package main

import "logger"

func main() {
    log := logger.GetLogger()
    log.SetLevel(logger.DEBUG)

    log.Debug("デバッグ情報")
    log.Info("処理を開始します")
    log.Error("エラーが発生しました")
}
```

## Goでのポイント

### `sync.Once` を使う理由

`sync.Once` は、指定した関数を**一度だけ**実行することを保証します。複数のゴルーチンから同時に呼び出されても安全です。

```go
// NGパターン：ダブルチェックロッキング（Goでは不要）
var mu sync.Mutex

func GetInstance() *Config {
    if instance == nil {  // この時点で複数のゴルーチンが通過する可能性
        mu.Lock()
        defer mu.Unlock()
        if instance == nil {
            instance = &Config{}
        }
    }
    return instance
}
```

`sync.Once` を使えば、上記のような複雑なロジックは不要です。

### テスタビリティの問題

Singletonはグローバル状態を持つため、テストが難しくなることがあります。テストのために、インターフェースを使う方法があります：

```go
// インターフェースを定義
type ConfigInterface interface {
    GetDatabaseURL() string
    IsDebug() bool
}

// Singletonはインターフェースを実装
func (c *Config) GetDatabaseURL() string {
    return c.DatabaseURL
}

func (c *Config) IsDebug() bool {
    return c.Debug
}

// 依存性注入でテスト可能に
type Service struct {
    config ConfigInterface
}

func NewService(config ConfigInterface) *Service {
    return &Service{config: config}
}
```

## 注意点

1. **過度な使用を避ける**: グローバル状態は依存関係を隠蔽し、テストを困難にします
2. **依存性注入を検討**: 可能であれば、Singletonよりも依存性注入を優先してください
3. **並行処理の考慮**: Goでは `sync.Once` を使えば安全に実装できます

## まとめ

- Singletonは、インスタンスを1つに制限するパターン
- Goでは `sync.Once` を使って安全に実装できる
- 便利だが、テスタビリティを考慮して使用を検討すべき

次章では、Factory Methodパターンを学びます。
