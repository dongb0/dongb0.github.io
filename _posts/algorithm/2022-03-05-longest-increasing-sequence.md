---
layout: post
title: "[动态规划笔记] 3 - 最长递增子序列"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - algorithm
---

[leetcode 300](1) 的经典动态规划问题，求一个数组中最长递增子序列的长度。

$O(n^2)$的解法：

1. dp数组元素dp\[i]表示第i个元素参与构成的最长递增序列的长度；
2. 状态转移方程：对于元素$j \subseteq [0, i-1] 且 nums[j] < nums[i]$，有 $dp[i] = max(dp[i], dp[j] + 1)$
3. 初始化：每个元素都能构成一个长度为1的递增子序列，因此dp数组全部初始化为1
4. 遍历顺序为从前往后扫描数组，每个数组元素再往前检查dp数组从中取最大值

据此可以写出第一版代码

```
func lengthOfLIS(nums []int) int {
    dp := make([]int, len(nums))
    for i := range dp {
        dp[i] = 1
    }
    ans := 1
    for i := 1; i < len(dp); i++ {
        for j := i - 1; j >= 0; j-- {
            if nums[i] > nums[j] {
                dp[i] = max(dp[i], dp[j] + 1)
            }
        }
        ans = max(ans, dp[i])
    }
    return ans
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```
--------

上面解法的内层循环中，我们每次还要往前遍历数组元素来找下标i能参与构成的最长递增序列，似乎还有优化空间。事实上最优的解法是$O(n×log(n))$复杂度，下面来看这是如何做的。

1. arr\[i]定义为*所有长度为i的递增子序列的最后一个元素中* **最小**的那个元素。比如对于数组\[2,3,5]，其中长度为2的子序列有 `{2, 3}, {2, 5}, {3, 5}`，最后一个元素分别有5和6，因此$arr[2]=min\{5, 6\} = 5$。arr中保存的就是所有最长递增子序列中字典序最小的那个，一旦碰到比当前序列最后一个元素要大的元素，我们肯定就能构造出一个更长的递增序列。
2. 在遍历原数组的过程中我们要不断更新arr中的元素，保持子序列是当前遇到过的所有序列中字典序最小的；因为arr中的元素是单调递增的，所以我们可以用二分查找来找到要更新的元素。

由此可以写出第二版$O(n×log(n))$的代码：
```
func lengthOfLIS(nums []int) int {
    dp := make([]int, len(nums))
    dp[0] = nums[0]
    size := 1
    for i := 1; i < len(nums); i++ {
        index := sort.Search(size, func (j int) bool { return dp[j] >= nums[i]})
        if index == size {
            dp[index] = nums[i]
            size += 1
        } else {
            dp[index] = nums[i]
        }
    }
    return size
}
```
这种解法应该不算动态规划了，只保存当前字典序最小的递增序列也许是一种贪心的思想。

-------

下面我稍微修改一下题目，需要输出长度最长的递增子序列，如果不唯一则输出多个。

在这个题目要求下$O(n×log(n))$的解法应当是做不到的，我们还得使用第一版$O(n^2)$的做法。只要每次碰到长度等于ans的序列，就记录下来；如果碰到更长的子序列，就把之前存的全部丢弃，重新开始记录即可。


// TODO: needs update 
```
func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func updateRes(arr [][][]int, oldLen, num int) [][][]int {
	index := oldLen - 1
	for _, oldRow := range arr[index] {
		if oldRow[len(oldRow) - 1] < num {	// TODO: remove duplicate
			newSeq := append(oldRow, []int{num}...)
			arr[index + 1] = append(arr[index + 1], newSeq)
		}
	}
	return arr
}

func lengthOfLIS(nums []int) [][]int {
	dp := make([]int, len(nums))
	dpRes := make([][][]int, len(nums))
	for i := range nums {
		dp[i] = 1
		dpRes[0] = append(dpRes[0], [][]int{{nums[i]}}...)
	}
	ans := 1
    for i := 1; i < len(dp); i++ {
        for j := i - 1; j >= 0; j-- {
            if nums[i] > nums[j] {
                dp[i] = max(dp[i], dp[j] + 1)
				dpRes = updateRes(dpRes, dp[j], nums[i])
            }
        }
        ans = max(ans, dp[i])
    }
	return dpRes[ans-1]
}


```

[1]: https://leetcode.com/problems/longest-increasing-subsequence/