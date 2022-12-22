# fmt

Go 打印 %v %+v %#v 的区别

1. **%v    只输出所有的值**

2. **%+v 先输出字段名字，再输出该字段的值**

3. **%#v 先输出结构体名字值，再输出结构体（字段名字+字段的值）**

```go
package main
import "fmt"

type student struct {
    id   int32
    name string
}

func main() {
    a := &student{id: 1, name: "xiaoming"}

    fmt.Printf("a=%v    \n", a)
    fmt.Printf("a=%+v    \n", a)
    fmt.Printf("a=%#v    \n", a) 
}

// 输出结果：-------------------------
a=&{1 xiaoming}
a=&{id:1 name:xiaoming}
a=&leetcode.student{id:1, name:"xiaoming"}
```

输出切片的时候，使用 %v !
