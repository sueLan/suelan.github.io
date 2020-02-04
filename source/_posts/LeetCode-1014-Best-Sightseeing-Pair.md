---
title: 'LeetCode:1014. Best Sightseeing Pair'
date: 2019-06-11 16:05:06
tags:
 - Algorithm 

categories: 
 - Algorithm

---

## 题目： [1014. Best Sightseeing Pair](https://leetcode.com/problems/best-sightseeing-pair/)
 

### 分析
这道题求 `the maximum score of a pair of sightseeing spots`; score = A[i] + A[j] + i - j)。 是个最优解问题。  
迭代中，当前idx为i; 要考虑 (0, i-1)的数中对score增益最大的数的下标是max_idx。 计算 (max_idx, i)的score， 并跟之前最大值做比较。

```c++
int maxScoreSightseeingPair(vector<int> & A) {
    // 2<= A.length <= 50000
    int max_idx = 0, res = A[0], inc = 0;
    for (int i = 1; i < A.size(); ++i) {
        inc = A[max_idx] - i + max_idx;
        res = max(res, A[i] + inc);
        max_idx = A[i] > inc ? i : max_idx;
    }

    return res;
}
```

**更为简约的写法：**
max_inc每次迭代后都自减1，是因为进入下一次迭代i+1的时候，它对分数的增益只剩下了A[max_inc] + i - (i+1)。
```c++
int maxScoreSightseeingPair2(vector<int>& A) {
    int max_inc = A[0] - 1, res = 0;
    for (int i = 1; i < A.size(); ++i, max_inc--) {
        res = max(res, max_inc + A[i]);
        max_inc = max(A[i], max_inc);
    }

    return res;
}
```

时间复杂度: $o(n)$