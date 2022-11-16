# 递归法

## 22 括号生成

[力扣](https://leetcode.cn/problems/generate-parentheses/)

题目描述：数字 `n` 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且 **有效的** 括号组合。

输入：n = 3
输出：["((()))","(()())","(())()","()(())","()()()"]

思路：这道题可以用递归+回溯的方法，满足（ 的数量不能大于n，（ 数量比 ）多的时候才能加右（ 。string长度等于2n的时候就递归到头了。优先向左括号增多的方向递归。

```go
backTracking = func(s *[]string, cur string, left int, right int, max int) {
		if len(cur) == max*2 { // 递归到头了
			*s = append(*s, cur)
		}

		// 这个遍历顺序类似于一个中序遍历，也满足了字典顺序，即把（ 看作1，）看作0，总是以一个较大的值排列
		// 左括号数量不能大于n
		if left < max {
			cur += "("
			backTracking(s, cur, left+1, right, max)
			cur = cur[:len(cur)-1]
		}
		// 左括号数量要大于右括号数量，才能网上加右括号
		if left > right {
			cur += ")"
			backTracking(s, cur, left, right+1, max)
			cur = cur[:len(cur)-1]
		}
	}
	backTracking(&res, "", 0, 0, n)
	return res 
}
```


