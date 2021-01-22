### 获取调用函数相关信息日志
## 注意
> 因获取函数信息，使用runtime.caller方法，增加开销，影响性能，非必要情况使用commlog记录日志。

#### 支持日志信息
1. 日志所在函数名，行号，文件位置
2. 日志所在函数的调用者函数名，行号，文件位置
3. 支持获取多级函数调用相关信息
3. 支持ip，日志等级，日志类型（自定义），日志信息（自定义）

#### 使用方法
```go
log.CallerLog(level, tag, msg, callerSkip)
```
```
level : 等级(支持Info,Warn,Error,Fatal),大小写严格约束使用上述字段(使用同commlog)
tag   : 标签(自定义，使用同commlog)
msg   : 信息(自定义，使用同commlog)
callerSkip   : 调用栈帧数（0为当前栈，即log.CallerLog,默认为1，获取业务代码函数调用者，应为>=2,超过栈帧数不输出日志）
```

#### 示例
```go
func Foo() {
	Bar()
}

func Bar() {
	log.CallerLog("Info","Bar","caller log", 2)
}

func main() {
	Foo()
    return
}

// output 
// INFO[0000] msg:caller log|funcName:main.Bar|filePath:/data1/www/htdocs/go/src/hot-dataservice/cmd/ptest.go|line:86|callerSkip:2|callFuncName:main.Foo|callFilePath:/data1/www/htdocs/go/src/hot-dataservice/cmd/ptest.go|callLine:82  cip=10.41.41.180 tag=Bar
```

