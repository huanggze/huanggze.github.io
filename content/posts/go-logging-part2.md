---
title: "Go 日志：klog"
date: 2022-04-07T16:19:16+08:00
toc: true
categories: ["go"]
series: ["golang-logging"]
---

zap 是 Uber 开源的 logging 库。

## Zap Logger

Zap 提供两种日志记录器（logger）：Sugared Logger 和 Logger。前者者支持结构化日志以及 Printf 风格日志，后者只支持结构化日志，但后者性能更好。

```go
// {"level":"info","ts":1649341600.931539,"caller":"awesomeProject/main.go:10","msg":"hello","val1":1,"val2":"two"}
logger, _ := zap.NewProduction()
logger.Info("hello",
    zap.Int("val1", 1),
    zap.String("val2", "two"),
)
```

```go
logger, _ := zap.NewProduction()
sugar := logger.Sugar()
// {"level":"info","ts":1649341600.931539,"caller":"awesomeProject/main.go:10","msg":"hello","val1":1,"val2":"two"}
sugar.Infow("hello", zap.Int("val1", 1), zap.String("val2", "two"))
// {"level":"info","ts":1649342764.54033,"caller":"awesomeProject/main.go:12","msg":"val1=1, val2=two"}
sugar.Infof("val1=%d, val2=%s", 1, "two")
```

Zap 支持三种方式创建预置 logger：NewExample()、NewProduction()、NewDevelopment()。它们的区别是日志信息内容。

## 定制 Logger

你也可以定制 logger（custom logger）来实现额外功能，比如输出日志到文件[^1]。此时，需要使用 New() 方法，而不是前面的 NewXXX()。New() 方法的函数签名如下。它需要接收一个 zapcore 参数。

```go
func New(core zapcore.Core, options ...Option) *Logger
```

配置 core 需要 Encoder，WriteSyncer，LogLevel：

- Encoder：如何写日志；
- WriteSyncer：日志写到哪；
- LogLevel：哪个级别的日志需要记录。

```go
// NewCore creates a Core that writes logs to a WriteSyncer.
func NewCore(enc Encoder, ws WriteSyncer, enab LevelEnabler) Core {
	return &ioCore{
		LevelEnabler: enab,
		enc:          enc,
		out:          ws,
	}
}
```

示例：
```go
func getEncoder() zapcore.Encoder {
    encoderConfig := zap.NewProductionEncoderConfig()
    encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
    encoderConfig.EncodeLevel = zapcore.CapitalLevelEncoder
    return zapcore.NewConsoleEncoder(encoderConfig)
}

func getLogWriter() zapcore.WriteSyncer {
	file, _ := os.Create("./test.log")
	return zapcore.AddSync(file)
}

// 日志输出到 test.log 文件
func main() {
	writeSyncer := getLogWriter()
	encoder := getEncoder()
	core := zapcore.NewCore(encoder, writeSyncer, zapcore.DebugLevel)

	logger := zap.New(core)
	// 2022-04-07T23:42:30.874+0800	INFO	hello	{"val": "1"}
	logger.Info("hello", zap.String("val", "1"))
}
```

## 日志轮转

Zap 不原生支持日志轮转[^2]，但可以接入第三方库来实现，比如 [lumberjack](gopkg.in/natefinch/lumberjack.v2) 作为 zapcore.WriteSyncer。

```go
import (
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"gopkg.in/natefinch/lumberjack.v2"
)

func getEncoder() zapcore.Encoder {
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
	encoderConfig.EncodeLevel = zapcore.CapitalLevelEncoder
	return zapcore.NewConsoleEncoder(encoderConfig)
}

// test.log
// test-2022-04-07T15-54-42.219.log <-- 轮转
func getLogWriter() zapcore.WriteSyncer {
	lumberJackLogger := &lumberjack.Logger{
		Filename:   "./test.log",
		MaxSize:    1,
		MaxBackups: 5,
		MaxAge:     30,
		Compress:   false,
	}
	return zapcore.AddSync(lumberJackLogger)
}

func main() {
	writeSyncer := getLogWriter()
	encoder := getEncoder()
	core := zapcore.NewCore(encoder, writeSyncer, zapcore.DebugLevel)

	logger := zap.New(core)
	logger.Info("hello", zap.String("val", "1"))
}
```

## 实现 go-logr/logr 接口

通过 zapr.NewLogger 接收一个 zap.Logger 参数，返回  logr.Logger 实现。

```go
log := zapr.NewLogger(zap.NewExample()).WithName("MyName")

// {"level":"info","logger":"MyName","msg":"hello","val1":1,"val2":{"k":1}}
// {"level":"debug","logger":"MyName","msg":"you should see this"}
// {"level":"error","logger":"MyName","msg":"uh oh","trouble":true,"reasons":[0.1,0.11,3.14]}
// {"level":"error","logger":"MyName","msg":"goodbye","code":-1,"error":"oops"}
log.Info("hello", "val1", 1, "val2", map[string]int{"k": 1})
log.V(1).Info("you should see this")
log.V(1).V(1).Info("you should NOT see this")
log.Error(nil, "uh oh", "trouble", true, "reasons", []float64{0.1, 0.11, 3.14})
log.Error(errors.New("oops"), "goodbye", "code", -1)

if zapLogger, ok := log.GetSink().(zapr.Underlier); ok {
    _ = zapLogger.GetUnderlying().Core().Sync()
}
```

[^1]: [Using zap log Library in go language project](https://developpaper.com/using-zap-log-library-in-go-language-project-translation/)
[^2]: [Zap FAQ](https://github.com/uber-go/zap/blob/master/FAQ.md#does-zap-support-log-rotation)