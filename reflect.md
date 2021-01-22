### golang反射

> 1. reflect.ValueOf()  获取输入参数的值
> 2. reflect.TypeOf()   获取输入参数的类型

```go

type S struct {
	Name string
	Age int
}

func main() {
	var s S
	s.Name = "test"
	fmt.Println(reflect.TypeOf(s.Name)) //output: string
	fmt.Println(reflect.ValueOf(s.Name)) // output: test
    return
}
```

#### struct的反射
```go 

type S struct {
	Name string
	Age int
}

func (a S) ATest() {
	fmt.Println("a is S struct")
}
func main() {
    s := S{Age:12, Name:"abc"}
    // 获取目标对象
	t := reflect.TypeOf(s)
    fmt.Println("type is:", t.Name())
    // 获取目标对象的值类型
    v := reflect.ValueOf(s)
    // numField()获取包含字段的总数
	for i:=0; i<t.NumField(); i++ {
        // 遍历包含的key
        key := t.Field(i)
        // 根据key获取对应的值
		value := v.Field(i).Interface()
		fmt.Printf("num %d is %s:%v = %v \n ", i+1, key.Name, key.Type, value)
    }
    // .numMethod获取S里面的方法
	for j:=0; j<t.NumMethod(); j++ {
		m := t.Method(j)
		fmt.Printf("num %d method is %s:%v \n", j+1, m.Name, m.Type)
	}
    return
    
}

// output :
// type is: S
// num 1 is Name:string = abc 
// num 2 is Age:int = 12 
// num 1 method is ATest:func(main.S) 

```

### 匿名或嵌入字段的反射(需继续理解)

```go

type S struct {
	Name string
	Age int
}

type Ss struct {
	S 	// 匿名字段
}

func (a S) ATest() {
	fmt.Println("a is S struct")
}
func main() {
	p := Ss{
		S{"SsName", 12},
	}

	t := reflect.TypeOf(p)
	// 添加#打印所有详情
	// Anonymous:true 标记为匿名字段
	fmt.Printf("%#v\n", t.Field(0))
	// 打印详情
	fmt.Printf("%#v\n", t.FieldByIndex([]int{0, 1}))

	// 获取匿名字段的值的详情
	v := reflect.ValueOf(p)
    fmt.Printf("%#v\n", v.Field(0))
}

// output :
// reflect.StructField{Name:"S", PkgPath:"", Type:(*reflect.rtype)(0x4aa020), Tag:"", Offset:0x0, Index:[]int{0}, Anonymous:true}
// reflect.StructField{Name:"Age", PkgPath:"", Type:(*reflect.rtype)(0x499ca0), Tag:"", Offset:0x10, Index:[]int{1}, Anonymous:false}
// main.S{Name:"SsName", Age:12}
```

#### 判断传入类型

```go 
func main() {
    s := S{Age: 12, Name:"test"}
	t := reflect.TypeOf(s)
	if t.Kind() == reflect.Struct {
		fmt.Println("ok")
	}
}
// output:
// ok
```

#### 通过反射修改内容
```go

type S struct {
	Name string
	Age int
}

type Ss struct {
	S 	// 匿名字段
}

func (a S) ATest() {
	fmt.Println("a is S struct")
}
func main() {
	s := &S{Age: 12, Name: "TEST"}
	v := reflect.ValueOf(s)

	// 修改值必须是指针
	if v.Kind() != reflect.Ptr {
		fmt.Println("not ptr")
		return
	}

	// 获取指针指向元素
	val := v.Elem()
	fmt.Println(val) // {TEST 12}

	age := val.FieldByName("Age")
	fmt.Println(age) // 12

	if age.Kind() == reflect.Int {
		age.SetInt(21)
	}
	fmt.Printf("%#v \n", *s) // main.S{Name:"TEST", Age:21}
    return
}
```

#### 通过反射调用函数
```go
func (s S) GetName(name string) {
	fmt.Println("my name is :", name)
}
func main() {
	s := S{Age:21,  Name: "test1"}
	v := reflect.ValueOf(s)
	funcName := v.MethodByName("GetName")

	args := []reflect.Value{reflect.ValueOf("test1")}

    funcName.Call(args)
}
// output:
// my name is : test1
```
> 1. 操作反射类型，要先确定反射类型，否则会导致panic
> 2. 反射主要与go interface类型相关