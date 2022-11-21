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

- 普通指针 <---------> unsafe.Pointer <--------> uintptr
