# Sort

## 排序任意切片

```go
sort.Slice(arr, func (i, j int) bool {
    return arr[i] < arr[j]
})

// [1,2,3,4,5] ints
// [a,b,c,d,e] bytes
// [自定义结构体切片排序]
```

## 排序任意数据结构

- 使用 `sort.Sort` 或者 `sort.Stable` 函数。
- 他们可以排序实现了 sort.Interface 接口的任意类型。

一个内置的排序算法需要知道三个东西：序列的长度，表示两个元素比较的结果，一种交换两个元素的方式；这就是 sort.Interface 的三个方法：

```go
type Interface interface {
    Len() int
    Less(i, j int) bool // i, j 是元素的索引
    Swap(i, j int)
}
```

```go
Less(i, j) bool
// 这个函数的意思是最后排序完：
// 第一个元素，第二个元素 ....
// 1,2,3是下标，表示第几个元素
// Less(1,2) = true; Less(2,3) = true ....
```

还是以上面的结构体切片为例子，我们为切片类型自定义一个类型名，然后在自定义的类型上实现 srot.Interface 接口

```go
type Person struct {
    Name string
    Age  int
}

// ByAge 通过对age排序实现了sort.Interface接口
type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

func main() {
    family := []Person{
        {"David", 2},
        {"Alice", 23},
        {"Eve", 2},
        {"Bob", 25},
    }
    sort.Sort(ByAge(family)) 
    // [{David, 2} {Eve 2} {Alice 23} {Bob 25}]
    fmt.Println(family)
}
```

实现了 sort.Interface 的具体类型不一定是切片类型；下面的 customSort 是一个结构体类型。

```go
type customSort struct {
    p    []*Person
    less func(x, y *Person) bool
}

func (x customSort) Len() int {len(x.p)}
func (x customSort) Less(i, j int) bool { return x.less(x.p[i], x.p[j]) }
func (x customSort) Swap(i, j int)      { x.p[i], x.p[j] = x.p[j], x.p[i] }
```

## 排序的时间复杂度

Go 的 sort 包中所有的排序算法在最坏的情况下会做 n log n 次 比较，n 是被排序序列的长度，所以排序的时间复杂度是 O(n log n)。其大多数的函数都是用改良后的快速排序算法实现的。
