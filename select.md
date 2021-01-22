### select 随机性
> select 随机性，随机选择一个可用通道做收发操作。
#### select会随机选择一个可用通道做收发操作。为什么呢？在源码中每一个select对应一个hselect结构,每个hselect结构下面都有个scase的数组记录每个case,在scase中记录着ch chan的结构也就是channel的结构pollorder将元素从新排列,scase就被乱序了
```go
func main() {
    c1 := make(chan int, 1)
    c2 := make(chan string, 1)

    c1 <- 1
    c2 <- "hi"

    select {
        case v1 := <- c1:
            fmt.Println(v1)
        case v2 := <- c2:
            fmt.Println(v2)
            panic("test")
    }
}

// output:
// 随机输出，有时1，有时panic

```



