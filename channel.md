### 读已关闭
```go
    c := make(int)
    c<-1
    close(c)
    fmt.Println(<-c)    // output 1
    fmt.Println(<-c)    // output 0
    fmt.Println(<-c)    // output 0

//string ： "test", "","" 
//struct : {aaa}, {}, {}
```
### 写已关闭
```go
    c := make(int)
    close(c)
    c<-1
// output
// panic: send on closed channel
```
### 源码
```go
// src/runtime/chan.go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

}
```

### 读写未初始化
```go
    var c chan int 
    //写
    c<-1                // goroutine 1 [chan send (nil chan)]:
    //读
    v， ok := <- c      // goroutine 1 [chan receive (nil chan)]:

```
> 读写如都是不阻塞情况下，认为chan = nil， 返回false \
> 如是阻塞，gopark，抛出相应异常。