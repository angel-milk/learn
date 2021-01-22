### go context
> context 请求全局上下文，携带截至时间，手动取消信号，包含并发安全的map用于携带数据。

#### 应用案例
1. 常用于多个goroutine数据交互
2. 超时控制
3. 上下文控制

#### context实现
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```
1. Deadline 返回time.Time,表示当前context应该结束的时间，ok表示有结束时间。
2. Done 当context被取消，或者超时返回的一个close的channel，告诉context要停止当前工作返回。
3. Err context取消原因。
4. Value context共享数据存储位置，协程安全。

#### context标准库
1. emptyCtx 
```go
type emptyCtx int
// 空context，实现函数返回nil，仅实现了context接口
```

```go
var (
    backgrounp = new(emptyCtx)
    todo       = new(emptyCtx)
)
```
* context.TODO context.Background 区别
> 源码注释：Background返回一个非nil的空上下文。它永远不会被取消，没有值，也没有截止日期。它通常用于主函数、初始化和测试，并作为传入请求的顶级上下文。TODO和Background作用一致，当不清楚上下文,或者上下文不可用时使用。


2. cancelCtx
```go
type cancelCtx struct {
    Context
    mu sync.Mutex
    Done chan struct{}
    children map[canceler]struct{}
    err error
}
// 继承context，实现canceler接口

```
* cancelCtx会主动进行取消，在自顶向下取消的过程中，会遍历children context，然后依次主动取消。


3. timerCtx
```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.
    deadline time.Time
}
// 继承cancelCtx，增加了timeout机制
```
* WithTimeout是通过WithDeadline来实现的，均对应timerCtx类型。通过parentCancelCtx函数的定义我们知道，timerCtx也会记录父子context关系。但是timerCtx是通过timer定时器触发cancel调用的

4. valueCtx
```go 
type valueCtx struct {
    Context
    key, val interface{}
}
// 存储键值对数据

```
#### context WithValue 传递参数

```go
func context1(ctx context.Context) {
	ctx = context.WithValue(ctx, "k1", "v1")
	context2(ctx)
}

func context2(ctx context.Context) {
	fmt.Println(ctx.Value("k1").(string))
}
func main() {
	ctx := context.Background()
	context1(ctx)
    return
}
// output : v1
```

* 如在context2赋值，在context1获取b不到值。context只能自上而下传递值。

#### value context 底层不是map
> 每一个单独的kv映射都对应一个valueCtx，想要传递多个值，就要建立多个valueCtx，这也就说明了不能自下向上传递原因。valueCtx的kv是通过接口形式实现的。
> 1. withvalue时，内部通过反射确定key，然后在赋值。（类似map）
> 2. value取值时,内部在本context查找，如果未找到，会在parent context递归查找，故不能自下向上传值。

### context传递方式
> 调用WithCancel，WithTimeOut, WithValue时处理父子context。内部通过propagateCancel实现。
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
// newCancelCtx是cancelCtx赋值父context的过程，而propagateCancel建立父子context之间的联系。
func propagateCancel(parent Context, child canceler) {
	if parent.Done() == nil {
		return // parent is never canceled
	}
	if p, ok := parentCancelCtx(parent); ok {// context包内部可以直接识别、处理的类型
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {// context包内部不能直接处理的类型，比如type A struct{context.Context},这种静默包含的方式
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
// 如果parent.done时nil，不做处理。
// parentCancelCtx根据parent context的类型，返回bool型ok，ok为真时需要建立parent对应的children，并保存parent->child映射关系(cancelCtx、timerCtx这两种类型会建立，valueCtx类型会一直向上寻找，而循环往上找是因为cancel是必须的，然后找一种最合理的)，这里children的key是canceler接口，并不能处理所有的外部类型，所以会有else，示例见上述代码注释处。对于其他外部类型，不建立直接的传递关系。

func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	for {
		switch c := parent.(type) {
		case *cancelCtx:
			return c, true
		case *timerCtx:
			return &c.cancelCtx, true
		case *valueCtx:
			parent = c.Context // 循环往上寻找
		default:
			return nil, false
		}
	}
}
```


#### notice
* cancel函数幂等，可以多次调用
* context中done channel被用于确认是否取消，通知取消。
* context只能自顶向下传值，反之则不可以。
* 如果有cancel，一定要保证调用，否则会造成资源泄露，比如timer泄露。
* context一定不能为nil，如果不确定，可以使用context.TODO()生成一个empty的context。
* Context要是全链路函数的第一个参数。 
```go 
func testContext(ctx context.Context) {

}
```

#### 示例
```go
func main() {

	ctx, cancel := context.WithCancel(context.Background())
	//ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	eatNum := eatH(ctx)
	defer cancel()
	for n := range eatNum {
		if n >= 10 {
			cancel()
			break
		}
	}
	fmt.Print("statistic result...\n")
	time.Sleep(1 * time.Second)
}

func eatH(ctx context.Context) <-chan int {
	c := make(chan int)
	n := 0
	t := 0
	go func() {
		for {
			select {
			case <-ctx.Done():
				fmt.Printf("cost time %d s, eat %d \n", t, n)
				return
			case c <- n:
				incr := rand.Intn(5)
				n += incr
				if n >= 10 {
					n = 10
				}
				t++
				fmt.Printf("i eat %d\n", n)
			}
		}

	}()

	return c
}
// output :
// i eat 1
// i eat 3
// i eat 5
// i eat 9
// i eat 10
// statistic result...
// i eat 10 需确认为啥再次打印
// cost time 6 s, eat 10 
```

```go
func main() {
	 // ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(10*time.Second))
	  ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	eatH1(ctx)
	defer cancel()

}

func eatH1(ctx context.Context) {
	n := 0
	for {
		select {
		case <-ctx.Done():
			fmt.Println("stop \n")
			return
		default:
			incr := rand.Intn(5)
			n += incr
			fmt.Printf("i eat %d \n", n)
		}
		time.Sleep(time.Second)
	}
}
// output
// i eat 1 
// i eat 3 
// i eat 5 
// i eat 9 
// i eat 10 
// i eat 13 
// i eat 13 
// i eat 13 
// i eat 14 
// i eat 14 
// stop 
```