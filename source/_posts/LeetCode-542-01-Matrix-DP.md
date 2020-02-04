---
title: 'LeetCode:542. 01 Matrix-DP'
date: 2019-05-23 10:02:45
tags:
 - Algorithm
categories: 
 - Algorithm
---
Given a matrix consists of 0 and 1, find the distance of the nearest 0 for each cell.

<!--more-->
 

## 题目: [01-matrix](https://leetcode.com/problems/01-matrix/)
Given a matrix consists of 0 and 1, find the distance of the nearest 0 for each cell.

The distance between two adjacent cells is 1.
```
Example:
Input:
[[0,0,0],
 [0,1,0],
 [1,1,1]]

Output:
[[0,0,0],
 [0,1,0],
 [1,2,1]]
```
## 分析： 
题目有点没讲明白，这道题，求值为1的cell到最近的0的最短距离。

这道题，我的第一想法是BFS，后来看到DP写法更优雅。[Simple-Java-solution](https://leetcode.com/problems/01-matrix/discuss/101051/Simple-Java-solution-beat-99-(use-DP))

可以发现规律。对matrix[i][j], 如果知道它上下左右四个cell到0的最短距离，那么
$$matrix[i][j] = min(left, top, right, bottom) + 1$$

1. 遍历matrix矩阵，matrix[i][j]不为0，计算 leftCell 跟 topCell最小值，再加1. 
2. 再倒叙遍历matrix, matrix[i][j]不为0, 先计算rightCell 跟 topCell的最小值，再跟min(left, top)的值比较

```c++
 vector<vector<int>> updateMatrix(vector<vector<int>>& matrix) {
     if (matrix.empty()) return {};
     int n = (int)matrix.size(), m = (int)matrix[0].size(), MAX_LEN = 10002;
     for (int i = 0; i < n; ++i) {
         for (int j = 0; j < m; ++j) {
             if (matrix[i][j] != 0) {
                 int top = (i - 1 < 0) ? MAX_LEN : matrix[i-1][j];
                 int left = (j - 1 < 0) ? MAX_LEN : matrix[i][j-1];
                 matrix[i][j] = 1 + min(top, left);
             }
         }
    
     for (int i = n - 1; i >= 0; i--) {
         for (int j = m - 1; j >= 0; j--) {
             if(matrix[i][j] != 0) {
                 int right = (j + 1 >= m) ? MAX_LEN: matrix[i][j+1];
                 int bottom = (i + 1 >= n) ? MAX_LEN: matrix[i+1][j];
                 matrix[i][j] = min(min(right, bottom) + 1, matrix[i][j]);
             }
         }
     }
     return matrix;
 }
```
### 时间复杂度 $o(n^2)$
### 空间复杂度 $o(1)$

```
Runtime: 184 ms, faster than 96.53% of C++ online submissions for 01 Matrix.
Memory Usage: 20.9 MB, less than 91.39% of C++ online submissions for 01 Matrix.
```
有人问，为啥要两次遍历？ 不能像下面这么写吗？

```c++
// 错误的写法
 vector<vector<int>> updateMatrix(vector<vector<int>>& matrix) {
       if (matrix.empty()) return {};
       int n = (int)matrix.size(), m = (int)matrix[0].size();
       for (int i = 0; i < n; ++i) {
           for (int j = 0; j < m; ++j) {
               if (matrix[i][j] != 0) {
                   int top = (i - 1 < 0) ? INT32_MAX : matrix[i-1][j];
                   int left = (j - 1 < 0) ? INT32_MAX : matrix[i][j-1];
                   int right = (j + 1 >= m) ? INT32_MAX : matrix[i][j+1];
                   int bottom = (i + 1 >= n) ? INT32_MAX : matrix[i+1][j];
                   matrix[i][j] = 1 + min(min(top, left), min(right, bottom));
               }
           }
       }
       return matrix;
   }
```
问题在于，顺序遍历时候，bottom跟right还没更新为正确的值。比如下图，遍历到matrix[3][0]的时候，matrix[3, 1] 还是1， 而matrix[2][0] == 2; 结果就出错了。
```
0 0 0 0 0 
1 0 0 0 0 
1 1 1 0 0 
1 1 1 1 0 
```
