---
title: 'LeetCode:413. Arithmetic Slices'
date: 2019-06-04 10:06:33
tags:
 - Algorithm 
categories: 
 - Algorithm 
---

##  [413. Arithmetic Slices](https://leetcode.com/problems/arithmetic-slices/)
这道题，简而言之，求数组里，等差序列的个数。 

比如
```
A = [1, 2, 3, 4]
有3个等差数列
[1, 2, 3], [2, 3, 4], [1, 2, 3, 4]
```

题目没有说明A是等差数列，所以也要考虑A不是等差数列，但是其子数组是等差数列的情况。
```
A = [1, 3, 5, 7, 9, 15, 20, 25]
1, 3, 5

3, 5, 7
1, 3, 5, 7

5, 7, 9
3, 5, 7, 9
1, 3, 5, 7, 9

15, 20, 25
```

## Solution 

```c++
int numberOfArithmeticSlices(vector<int>& A) {
    int dp = 0;
    int sum = 0;

    for (int i = 2; i < A.size(); ++i) {
        if (A[i] - A[i-1] == A[i-1] - A[i-2])  {
           dp = 1+dp;
           sum += dp;
        } else {
           dp = 0;
        }
    }
    return sum;
}
```