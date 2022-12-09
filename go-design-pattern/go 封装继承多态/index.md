# Go 封装继承多态

## 继承

```go
type Animal struct {
    ...
}

func (a *Animal) eat() {
    ...
}

type Cat struct {
    Animal // 继承
    ...
}

func main() {
    cat := Cat{}
    cat.eat()
}
```

## 封装

```go
type Animal struct {
    ...
}

type AnimalInterface interface {
    eat()
}

// 实现了 AnimalInterface 接口
func (a *Animal) eat() {
    ...
}

func Cat struct {
    AnimalInterface
    ...
}

func main() {
    cat := Cat{}
    cat.eat() // 调用 AnimalInterface 的方法
}
```

## 多态

```go
type AnimalInterface struct {
    eat()
}

type Cat struct {
    ...
}

func (c *Cat) eat() {
    ...
}

type Dog struct {
    ...
}

type (d *Dog) eat() {
    ...
}

func main() {
    var AnimalInterface dog = Dog{}
    var AnimalInterface cat = Cat{}
    dog.eat()
    cat.eat()
}
```

## 总结：少用继承，多用组合加委托！
