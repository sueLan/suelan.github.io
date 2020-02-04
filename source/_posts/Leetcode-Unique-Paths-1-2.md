---
title: LeetCode Unique Paths 1-2
date: 2019-05-08 20:57:05
categories: 
    - Algorithm
tags:
    - Algorithm
---
Leetcode Unique Paths 1-2 
<!-- more --> 

# Unique Paths 1-2

## 题目1： [Unique Paths 1](https://leetcode.com/problems/unique-paths/)

![e2ca2848.png](/img//c5619f25-13a6-4e38-b7f8-d87ff624f5b5/e2ca2848.png)

### 动态规划解题步骤要点: 

1. 找到最优解的结构
2. 递归方案
3. bottom-up 或者 top-down方式求最优解
4. 优化，空间换时间

### 分析

这道题很简单。
1. 定义一个 mxn 的二维数组grides[m][n]， 对应格子地图。对应途中grides[2][6]

![e594b650.png](/img//c5619f25-13a6-4e38-b7f8-d87ff624f5b5/e594b650.png)
比如，我们想知道从(0,0)到(1, 2)有几条路。根据题目要求，只能往右走或者下走一个格子。
我们先看从(1, 1)到（1，2），两种走法： 
- (2, 0) -> (1, 2)
- (1, 1) -> (1, 2)

再看从(0, 0)到(2， 0) 只有一条路: (0, 0) -> (1, 1) -> (2, 0)； grides[2][0] = 1。

从(0, 0)格子到(1,1)格子，有两种路: 
- (0,0) -> (0, 1)-> (1, 1)
- (0,0) -> (1, 0)-> (1, 1)
那么grides(1, 1) = 2. 
 
很容易发现规律， grids[m][n] = grids[m - 1][n] + grides[m][n -1]; 可用递归求解。但是纯碎的递归有很多重复计算。 如下图的递归树： 
![72a3d4ef.png](/img//c5619f25-13a6-4e38-b7f8-d87ff624f5b5/72a3d4ef.png)
所以我们引入一个record数组，记录已经计算的结果，省去重复计算。

时间复杂度: 
o(m* n)

```c++
int inner_path(int i, int j, int m, int n, vector<vector<int>>& record) {
    if (i < 0 || j < 0) {
        return 0;
    }

    if (record[i][j]) {
        return record[i][j];
    }

    if ((i == 0) && (j == 0)) {
        return 1;
    }

    record[i][j] = inner_path(i-1, j, m, n, record) + inner_path(i, j-1, m, n, record);
    return record[i][j];
}

int uniquePaths(int m, int n) {
    vector<vector<int>> res(m, vector<int>(n, 0));
    return inner_path(m-1, n-1, m, n, res);
}
```

### [Unique Paths 2](https://leetcode.com/problems/unique-paths-ii/)

分析： 
跟Unique Paths 1比起来，多了障碍物。如果(i, j)是障碍物，那么grid(i, j) = 0, 表示我们无法经由(i, j) 到达终点。



```c++
int path_helper(int i, int j, int m, int n, vector<vector<int>>& record, vector<vector<int>> &obstacleGrid) {
        if (i < 0 || j < 0 || obstacleGrid[i][j] == 1) {
            return 0;
        }

        if (record[i][j]) {
            return record[i][j];
        }

        if ((i == 0) && (j == 0)) {
            return 1;
        }

        record[i][j] = path_helper(i - 1, j, m, n, record, obstacleGrid) + path_helper(i, j - 1, m, n, record, obstacleGrid);
        return record[i][j];
    }

    int uniquePathsWithObstacles(vector<vector<int>>& obstacleGrid) {
        if (obstacleGrid.empty()) return 0;
        int m = (int)obstacleGrid.size();
        int n = (int)obstacleGrid[0].size();

        vector<vector<int>> res(m, vector<int>(n, 0));
        return path_helper(m - 1, n - 1, m, n, res, obstacleGrid);
    }
```

自底向上的写法： 

```c++
   int uniquePathsWithObstacles(vector<vector<int>>& grid) {
        int rows = grid.size();
        if(rows == 0) return 0;
        int cols = grid[0].size();
        vector<vector<long>> res(rows+1, vector<long>(cols+1, 0));
        for(int i=rows-1; i>=0;i--){
            for(int j=cols-1; j>=0;j--){
                if(grid[i][j] == 1) res[i][j] = 0;
                else if(i == rows-1 && j == cols-1) res[i][j] = 1;
                else res[i][j] = res[i][j+1] + res[i+1][j];
            }
        }
        return res[0][0];
    }
```