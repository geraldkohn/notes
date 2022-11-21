## 1 命令

### go run/go build/go install

**go run**

go run 编译并直接运行程序，它会产生一个临时文件。

**go build**

go build 用于测试编译包，主要用来检查是否有编译错误，如果是一个可执行文件的源码（即是main包），就会在当前目录直接生成一个可执行文件。

**go install**

go install 的作用主要有两步：

1. 编译导入的包文件，所有导入的包文件编译完才会编译主程序。

2. 将编译后生成的可执行文件放到 bin 目录下（GOPATH/bin），编译后的包文件放到 pkg 目录下（GOPATH/pkg）。（GOPATH为Go的工作目录）

## 2 Slice

### slice 和数组的区别？

数组是定长的，长度定义好之后就不能更改了。slice 是可以动态扩容的。slice 是一个结构体，包括三个字段：长度，容量，一个指向底层数组的指针。

```go
type slice struct {
    array unsafe.Pointer // 元素指针
    len   int // 长度 
    cap   int // 容量
}
```

### slice 扩容策略？

当slice中的长度等于容量的时候，再向slice中追加元素的时候，就会发生扩容。扩容遵循如下规则：

* 如果原来的切片容量小于1024，则新的切片容量扩大到原来的2倍。
* 如果原来的切片容量大于等于1024，则新的切片容量扩大到原来的1.25倍。
* 如果扩容后的大小仍然不满足，那么直接扩容到所需要的容量。

### slice 作为函数参数？

## uintptr & unsafe.Pointer

### unsafe.Pointer

从名字来看它是不安全的和指针相关，unsafer.pointer主要的功能就是不同类型指针间的转换。

```go
func main() {
   var a *int8
   var b *int16
   a = new(int8)
   b = new(int16)
   *b = 10
   *a = *b
   fmt.Println(a)
}
```

1. 初始化一个指针类型的int8 a
2. 初始化一个指针类型的int16 b
3. 给指针b指向的内存赋值10
4. 指针a内存赋值指针b的内存值

结果：`cannot use *b (type int16) as type int8 in assignment` 

go是强类型语言，这种即使都是int，这样转换也是不行的，于是出现了unsafe.pointer。

```go
var a *int8
var b *int16
a = new(int8)
b = new(int16)
*b = 10
upb := unsafe.Pointer(b)
b_int8ptr := (*int8)(upb)
*a = *(b_int8ptr)
fmt.Println(*a)
```

- 初始化一个指针类型的int8 a
- 初始化一个指针类型的int16 b
- 给指针b指向的内存赋值10
- 通过unsafe.pointer获取b的指针`upb`
- 把upb转成*int8指针 `b_int8ptr`
- 获取b_int8ptr的内存值赋值给a的地址指向的空间
  结果：`10`  
  unsafer.pointer可以转换不同类型的指针。

### uintptr

uintptr 实际上就是用一个 uint 来表示地址的。

```go
var a, b uintptr
a = 10
b = 10
fmt.Println(a + b)
```

地址不能直接相加，即使将指针转换为 unsafe.Pointer 也不能直接相加。

```go
var a, b *int
a, b = new(int), new(int)
c := unsafe.Pointer(a) + unsafe.Pointer(b)
// invalid operation: unsafe.Pointer(a) + unsafe.Pointer(b) (operator + not defined on unsafe.Pointer)

var a, b *int
a, b = new(int), new(int)
c := a + b  
//invalid operation: a + b (operator + not defined on pointer)
```

uintptr 可以做地址之间的运算

```go
var a, b *int
a, b = new(int), new(int)
c := uintptr(unsafe.Pointer(a)) + uintptr(unsafe.Pointer(b))
fmt.Println(c)
```

### unsafe.Pointer 和 uintptr 的一些组合操作

```go
type User struct {
   Name string
   Age  int8
}
func main() {
   u := &User{}
   address := unsafe.Pointer(u)
   ageOffset := unsafe.Offsetof(u.Age)
   agePtr := unsafe.Pointer(uintptr(uAddress) + ageOffset)
   *((*int)(agePtr)) = 10
   fmt.Println(u.Age)
}
```

- 通过unsafe.Pointer获取u的地址uAddress
- 通过unsafe.Offsetof获取u.age的偏移量
- uintptr转换uAddress 加上age的偏移量就找到了age的地址agePtr
- agePtr转成指针型int，然后取其内存空间，赋值10

结合unsafe和uintptr也可以对某些变量赋值，整个流程看下来比较复杂。

### 总结

- unsafe.Pointer只是单纯的通用指针类型，用于转换不同类型指针，它不可以参与指针运算；

- 而uintptr是用于指针运算的，GC 不把 uintptr 当指针，也就是说 uintptr 无法持有对象， uintptr 类型的目标会被回收；

- unsafe.Pointer 可以和 普通指针 进行相互转换；

- unsafe.Pointer 可以和 uintptr 进行相互转换。
* 普通指针 <---------> unsafe.Pointer <--------> uintptr
