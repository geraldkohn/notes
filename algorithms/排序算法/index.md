# 排序算法

## 1 冒泡排序

冒泡排序（Bubble Sort）也是一种简单直观的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢"浮"到数列的顶端。

![](./images/bubbleSort.gif)

```go
unc bubbleSort(arr []int) []int {
    length := len(arr)
    for i := 0; i < length; i++ {
        for j := 0; j < length-1-i; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
            }
        }
    }
    return arr
}
```

## 2 归并排序

时间复杂度：O(nlogn)

空间复杂度：O(n)

算法步骤：

1. 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列；

2. 设定两个指针，最初位置分别为两个已经排序序列的起始位置；

3. 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置；

4. 重复步骤 3 直到某一指针达到序列尾；

5. 将另一序列剩下的所有元素直接复制到合并序列尾。

![](./images/mergeSort.png)

```go
func mergeSort(arr []int) []int {
    length := len(arr)
    if length < 2 {
        return arr
    }
    middle := length / 2
    left := arr[0:middle]
    right := arr[middle:]
    return merge(mergeSort(left), mergeSort(right))
}

func merge(left []int, right []int) []int {
    var result []int
    for len(left) != 0 && len(right) != 0 {
        if left[0] <= right[0] {
            result = append(result, left[0])
            left = left[1:]
        } else {
            result = append(result, right[0])
            right = right[1:]
        }
    }

    for len(left) != 0 {
        result = append(result, left[0])
        left = left[1:]
    }

    for len(right) != 0 {
        result = append(result, right[0])
        right = right[1:]
    }

    return result
}
```

## 3 快速排序

在平均状况下，排序 n 个元素要 Ο(nlogn) 次比较。在最坏状况下则需要 Ο(n2) 次比较，但这种状况并不常见。事实上，快速排序通常明显比其他 Ο(nlogn) 算法更快，因为它的内部循环（inner loop）可以在大部分的架构上很有效率地被实现出来。

算法步骤：

1. 从数列中挑出一个元素，称为 "基准"（pivot）;

2. 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区（partition）操作；

3. 递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序；

```go
// func debug(arr []int, left int, right int) {
//    fmt.Printf("left: %d, right: %d, arr: %v\n", left, right, arr)
// }

func quickSort(arr []int) {
    _quickSort(arr, 0, len(arr)-1)
}

func _quickSort(arr []int, left int, right int) {
    if left < right {
        // 位置划分
        middleIndex := partition(arr, left, right)
        // 左面排序
        _quickSort(arr, left, middleIndex-1)
        // 右面排序
        _quickSort(arr, middleIndex+1, right)
    }
}

// 标准分割函数（分区）
func partition(arr []int, left int, right int) int {
    // debug
//    fmt.Printf("进入一次划分: left: %d, right: %d, arr: %v\n", left, right, arr)

    pivot := arr[left] // 导致 left 位置为空
//     arr[left] = 0
    for left < right {
        // right 指针值 >= pivot，right 指针左移
        for left < right && arr[right] >= pivot {
            right--
        }
        // 填补 left 位置的值, arr[left] 是一个空位置
        arr[left] = arr[right]
        // debug
//        arr[right] = 0
//        debug(arr, left, right)

        // left 指针值 <= pivot left 指针右移
        for left < right && arr[left] <= pivot {
            left++
        }
        // 填补 right 位置的值, arr[right] 是一个空位置
        arr[right] = arr[left]
        // debug
//        arr[left] = 0
//        debug(arr, left, right)
    }
    // left = right
    arr[left] = pivot
    return left
}
```

```bash
[]int{3,3,6,2,1,2,3,3,7}

进入一次划分: left: 0, right: 8, arr: [3 3 6 2 1 2 3 3 7]
left: 0, right: 5, arr: [2 3 6 2 1 0 3 3 7]
left: 2, right: 5, arr: [2 3 0 2 1 6 3 3 7]
left: 2, right: 4, arr: [2 3 1 2 0 6 3 3 7]
left: 4, right: 4, arr: [2 3 1 2 0 6 3 3 7]
进入一次划分: left: 0, right: 3, arr: [2 3 1 2 3 6 3 3 7]
left: 0, right: 2, arr: [1 3 0 2 3 6 3 3 7]
left: 1, right: 2, arr: [1 0 3 2 3 6 3 3 7]
left: 1, right: 1, arr: [1 0 3 2 3 6 3 3 7]
left: 1, right: 1, arr: [1 0 3 2 3 6 3 3 7]
进入一次划分: left: 2, right: 3, arr: [1 2 3 2 3 6 3 3 7]
left: 2, right: 3, arr: [1 2 2 0 3 6 3 3 7]
left: 3, right: 3, arr: [1 2 2 0 3 6 3 3 7]
进入一次划分: left: 5, right: 8, arr: [1 2 2 3 3 6 3 3 7]
left: 5, right: 7, arr: [1 2 2 3 3 3 3 0 7]
left: 7, right: 7, arr: [1 2 2 3 3 3 3 0 7]
进入一次划分: left: 5, right: 6, arr: [1 2 2 3 3 3 3 6 7]
left: 5, right: 5, arr: [1 2 2 3 3 0 3 6 7]
left: 5, right: 5, arr: [1 2 2 3 3 0 3 6 7]
[1 2 2 3 3 3 3 6 7]
```

## 4 堆排序

堆排序（Heapsort）是指利用堆这种数据结构所设计的一种排序算法。堆积是一个近似完全二叉树的结构，并同时满足堆积的性质：即子结点的键值或索引总是小于（或者大于）它的父节点。堆排序可以说是一种利用堆的概念来排序的选择排序。分为两种方法：

1. 大顶堆：每个节点的值都大于或等于其子节点的值，在堆排序算法中用于升序排列；
2. 小顶堆：每个节点的值都小于或等于其子节点的值，在堆排序算法中用于降序排列；

堆排序的平均时间复杂度为 Ο(nlogn)。

**堆排序的基本思想是：将待排序序列构造成一个大顶堆，此时，整个序列的最大值就是堆顶的根节点。将其与末尾元素进行交换，此时末尾就为最大值。然后将剩余n-1个元素重新构造成一个堆，这样会得到n个元素的次小值。如此反复执行，便能得到一个有序序列了**
