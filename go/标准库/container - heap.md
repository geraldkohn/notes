## heap.Interface 接口

```go
// heap.Interface
type Interface interface {
    sort.Interface
    // 向末尾添加元素
    Push(x interface{}) // add x as element Len()
    // 从末尾删除元素
    Pop() interface{}   // remove and return element Len() - 1.
}

// sort.Interface
type Interface interface {
    // Len 方法返回集合中的元素个数
    Len() int
    // Less 方法报告索引 i 的元素是否比索引 j 的元素小
    Less(i, j int) bool
    // Swap 方法交换索引 i 和 j 的两个元素
    Swap(i, j int)
}
```

可以看到，`heap.Interface`接口组合了`sort.Interface`接口，所以任意类型想要实现`heap.Interface`接口，不但要实现`Push()/Pop()`方法，还需要实现`Len()/Less()/Swap()`方法。

## heap 提供的方法

```go
// 一个堆在使用任何堆操作之前应先初始化。Init 函数对于堆的约束性是幂等的（多次执行无意义），
// 并可能在任何时候堆的约束性被破坏时被调用。本函数复杂度为 O(n)，其中 n 等于 h.Len()
func Init(h Interface)

// 向堆 h 中插入元素 x，并保持堆的约束性。复杂度 O(log(n))，其中 n 等于 h.Len()
func Push(h Interface, x interface{})

// 删除并返回堆 h 中的最小元素（不影响约束性）。复杂度 O(log(n))，其中 n 等于 h.Len()
// 等价于 Remove(h, 0)
func Pop(h Interface) interface{}

// 删除堆中的第 i 个元素，并保持堆的约束性。复杂度 O(log(n))，其中 n 等于 h.Len()
func Remove(h Interface, i int) interface{}

// 在修改第 i 个元素后，调用本函数对堆进行再平衡，比删除第 i 个元素后插入新元素更有效率
// 复杂度 O(log(n))，其中 n 等于 h.Len()
func Fix(h Interface, i int)
```

## heap 使用

### 大顶堆/小顶堆

```go
package main

import (
    "container/heap"
    "fmt"
)

// IntHeap 实现 heap.Interface 接口
type IntHeap []int

func (h IntHeap) Len() int {
    return len(h)
}

func (h IntHeap) Less(i, j int) bool {
    // 如果设置 h[i] < h[j] 就是小顶堆，h[i] > h[j] 就是大顶堆
    return h[i] < h[j]
}

func (h IntHeap) Swap(i, j int) {
    h[i], h[j] = h[j], h[i]
}

func (h *IntHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
    x := (*h)[len(*h)-1]
    *h = (*h)[:len(*h)-1]
    return x
}

func main() {
    // 创建切片
    h := &IntHeap{2, 1, 5, 6, 4, 3, 7, 9, 8, 0}

    // 初始化小顶堆
    heap.Init(h)
    fmt.Println(*h) // [0 1 3 6 2 5 7 9 8 4]

    // Pop 元素
    fmt.Println(heap.Pop(h).(int)) // 0

    // Push 元素
    heap.Push(h, 6)
    fmt.Println(*h) // [1 2 3 6 4 5 7 9 8 6]

    for h.Len() != 0 {
        fmt.Printf("%d ", heap.Pop(h).(int)) // 1 2 3 4 5 6 6 7 8 9
    }
}
```

### 优先级队列

```go
package main

import (
    "container/heap"
    "fmt"
)

// Entry 是 priorityQueue 中的元素
type Entry struct {
    // value 是 Entry 中的数据，可以是任意类型，这里使用 string
    value string
    // priority 定义 Entry 在 priorityQueue 中的优先级
    priority int
    // index 是 Entry 在 heap 中的索引位置
    // Entry 加入 Priority Queue 后， Priority 会变化时，很有用
    // 如果 Entry.priority 一直不变的话，可以删除 index
    index int
}

// PriorityQueue 实现 heap.Interface 接口方法
type PriorityQueue []*Entry

func (pq PriorityQueue) Len() int {
    return len(pq)
}

func (pq PriorityQueue) Less(i, j int) bool {
    // 这里生成小根堆
    return pq[i].priority < pq[j].priority
}

func (pq PriorityQueue) Swap(i, j int) {
    pq[i], pq[j] = pq[j], pq[i]
    pq[i].index, pq[j].index = i, j
}

// Push 往 priorityQueue 中放 Entry
func (pq *PriorityQueue) Push(x interface{}) {
    entry := x.(*Entry)
    entry.index = len(*pq)
    *pq = append(*pq, entry)
}

// Pop 从 priorityQueue 中取出最优先的 Entry
func (pq *PriorityQueue) Pop() interface{} {
    x := (*pq)[len(*pq)-1]
    x.index = -1 // for safety
    *pq = (*pq)[:len(*pq)-1]
    return x
}

// update 更改 Entry 在 priorityQueue 的值和优先级
func (pq *PriorityQueue) update(entry *Entry, value string, priority int) {
    entry.value = value
    entry.priority = priority
    heap.Fix(pq, entry.index)
}

func main() {
    pq := &PriorityQueue{
        {value: "Alice", priority: 5},
        {value: "Bob", priority: 2},
        {value: "Allen", priority: 4},
        {value: "An", priority: 1},
        {value: "Angel", priority: 3},
    }

    // 初始化优先级队列 pq
    heap.Init(pq)
    // 往 pq 中添加 entry
    entry := &Entry{value: "Claudia", priority: 0}
    heap.Push(pq, entry)
    // 更改 entry 的优先级
    pq.update(entry, entry.value, 6)
    for pq.Len() != 0 {
        entry := heap.Pop(pq).(*Entry)
        fmt.Printf("priority:%d, value:%s\n", entry.priority, entry.value)
    }
}

// 控制台输出：
// priority:1, value:An
// priority:2, value:Bob
// priority:3, value:Angel
// priority:4, value:Allen
// priority:5, value:Alice
// priority:6, value:Claudia
```

## 底层实现分析

### init 函数

```go
// Init establishes the heap invariants required by the other routines in this package.
// Init is idempotent with respect to the heap invariants
// and may be called whenever the heap invariants may have been invalidated.
// The complexity is O(n) where n = h.Len().
func Init(h Interface) {
    // heapify
    n := h.Len()
    for i := n/2 - 1; i >= 0; i-- { // 初始化堆
        down(h, i, n) // 堆的下沉函数
    }
}

func down(h Interface, i0, n int) bool { // 堆的下沉函数
    i := i0
    for {
        j1 := 2*i + 1
        if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
            break
        }
        j := j1 // left child
        if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
            j = j2 // = 2*i + 2  // right child
        }
        if !h.Less(j, i) {
            break
        }
        h.Swap(i, j)
        i = j
    }
    return i > i0
}
```

### Push 函数

```go
// Push 通过将新元素 x 追加到集合的尾部，然后将元素 x 上浮平衡堆
// Push pushes the element x onto the heap.
// The complexity is O(log n) where n = h.Len().
func Push(h Interface, x interface{}) {
    // 这里的 Push(x) 就是 heap.Interface 接口的 Push 方法，将新元素 x 追加到集合的尾部
    h.Push(x)
    up(h, h.Len()-1) // 堆的上浮函数
}

func up(h Interface, j int) { // 堆的上浮函数
    for {
        i := (j - 1) / 2 // parent
        if i == j || !h.Less(j, i) {
            break
        }
        h.Swap(i, j)
        j = i
    }
}
```

### Pop 函数

```go
// 将最小元素与尾部元素的进行位置交换，然后通过下沉函数平衡堆，最后将下标在尾部的最小元素移除
// Pop removes and returns the minimum element (according to Less) from the heap.
// The complexity is O(log n) where n = h.Len().
// Pop is equivalent to Remove(h, 0).
func Pop(h Interface) interface{} {
    n := h.Len() - 1
    h.Swap(0, n) // 将最小元素和末尾元素位置交换
    down(h, 0, n) // 堆的下沉函数
    // 这里的 Pop(x) 就是 heap.Interface 接口的 Pop 方法，将下标在尾部的最小元素移除
    return h.Pop() 
}
```

### Remove 函数

```go
// 将下标为 i 的元素与尾部元素的进行位置交换，然后通过下沉函数平衡堆，最后将下标在尾部的元素移除
// Remove removes and returns the element at index i from the heap.
// The complexity is O(log n) where n = h.Len().
func Remove(h Interface, i int) interface{} {
    n := h.Len() - 1
    if n != i {
        h.Swap(i, n)
        if !down(h, i, n) {
            up(h, i)
        }
    }
    return h.Pop()
}
```

### Fix 函数

```go
// 在修改第 i 个元素后，调用本函数对堆进行再平衡
// Fix re-establishes the heap ordering after the element at index i has changed its value.
// Changing the value of the element at index i and then calling Fix is equivalent to,
// but less expensive than, calling Remove(h, i) followed by a Push of the new value.
// The complexity is O(log n) where n = h.Len().
func Fix(h Interface, i int) {
    if !down(h, i, h.Len()) {
        up(h, i)
    }
}
```

## 参考：

1. [GO语言heap剖析及利用heap实现优先级队列](https://www.cnblogs.com/huxianglin/p/6925119.html)
2. [GO语言实现堆、栈、队列、优先级队列](https://www.jianshu.com/p/15e6d436d471)


