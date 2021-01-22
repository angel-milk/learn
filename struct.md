### struct加tag区别

```go
type J struct {
	a string
	b string  `json:"B"`
	C string
	D string  `json:"WITH"`
}


func main() {
	j := J {
		a:"1",
		b:"2",
		C:"3",
		D:"4",
	}
	json , _ := jsoniter.Marshal(j)
	fmt.Println(j)
	fmt.Println(string(json))
    return
}
// output 
// {1 2 3 4}
// {"C":"3","WITH":"4"}
```
> 结构体里定义了四个字段，分别对应 小写无tag，小写+tag，大写无tag，大写+tag。\
转为json后首字母小写的不管加不加tag都不能转为json里的内容，而大写的加了tag可以取别名，不加tag则json内的字段跟结构体字段原名一致。

* json包里使用的时候，结构体里的变量不加tag能不能正常转成json里的字段？


1. 如果变量首字母小写，则为private。无论如何不能转，因为取不到反射信息。

2. 如果变量首字母大写，则为public。
    1.  不加tag，可以正常转为json里的字段，json内字段名跟结构体内字段原名一致。
    2. 加了tag，从struct转json的时候，json的字段名就是tag里的字段名，原字段名已经没用。