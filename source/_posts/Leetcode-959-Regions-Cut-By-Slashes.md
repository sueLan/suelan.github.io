---
title: LeetCode 959. Regions Cut By Slashes 笔记
date: 2019-04-23 10:23:33
tags:
    - Algorithm
categories:
    - Algorithm
---

题目：https://leetcode.com/problems/regions-cut-by-slashes/description/

## Solution1: DFS 

Time Complexity::  $O(n^2)$

### 分析： 
[这个神奇的思路](https://leetcode.com/problems/regions-cut-by-slashes/discuss/205674/C++-with-picture-DFS-on-upscaled-grid)分割格子，转化为图的DFS问题。
```json
Input:
[
  "\\/",
  "/\\"
]
```

把`\\`，`/`或者` `分别分割成3*3的格子。图转化为: 

```json
[
 1,0,0,0,0,1,
 0,1,0,0,1,0,
 0,0,1,1,0,0
 0,0,1,1,0,0
 0,1,0,0,1,0
 1,0,0,0,0,1
]
```

问题就变成： 计算被1分割开的区域的数量。

类似的[孤岛数量问题](https://www.geeksforgeeks.org/find-number-of-islands/) 都是同一个问题的变种：[Counting the number of connected components in an undirected graph](https://www.geeksforgeeks.org/connected-components-in-an-undirected-graph/)

```c++
void dfs(vector<vector<int>>& board, int i, int j ) {
    // index out of range
    if (i < 0 || j < 0 || i >= board.size() || j >= board[0].size()) return;
    // this grid has been visited or it was part of a slash character
    if (board[i][j] == 1) return;

    // mark this grid visited
    board[i][j] = true;

    dfs(board, i - 1, j);
    dfs(board, i + 1, j);
    dfs(board, i, j - 1);
    dfs(board, i, j + 1);
}

int regionsBySlashes(vector<string>& grid) {
    if (grid.empty())
        return 0;
    int row = (int)grid.size(), col = (int)grid[0].size();
    vector<vector<int>> board (row * 3, vector<int>(col * 3, 0));

    // n*n graph represented as 3n*3n graph
    for (int i = 0; i < row; ++i) {
        for (int j = 0; j < col; ++j) {
            if (grid[i][j] == '/') board[i * 3][j * 3 + 2] = board[i * 3 + 1][j * 3 + 1] = board[i * 3 + 2][j * 3] = 1;
            if (grid[i][j] == '\\') board[i * 3][j * 3] = board[i * 3 + 1][j * 3 + 1] = board[i * 3 + 2][j * 3 + 2] = 1;
        }
    }

    int cnt = 0;
    for (int i = 0; i < row * 3; ++i) {
        for (int j = 0; j < col * 3; ++j) {
            // only count components connected by space
            if (!board[i][j]) {
               dfs(board, i, j);
               cnt++;
            }
        }
    }

    return cnt;
}

```

如果分割为2*2的格子，会遇到这个问题:
```
Input:
[
  "//",
  "/ "
]
Output: 5
Expected: 3

0101
1010
0100
1000
```
01**0**1
1**0**10
**0**100
1000

加粗的这三个0，被分割开了。题意要求是连在一起的。


## Solution2: DSU

Time Complexity:  $O(n^2*\alpha(n))$
Space Complexity: $O(n^2)$

### 分析： 

#### DSU: 
https://www.youtube.com/watch?v=YKE4Vd1ysPI
https://www.youtube.com/watch?v=gpmOaSBcbYA

DSU中用数组来表示树， 如下： 

| idx | 0 |  1 | 2 |
| --- | --- | --- | --- |
| parent | 1 | -1 | 1 |

 
parent[0] = 1, 代表node 0的parent是1； parent[2] = 1代表node 2的parent是1; parent[1] = -1, 代表它是root。 

```
   1
 /  \
0    2
```

两个operation： 
**find(x)**: find root of cluster in which x is 

```
   1                   5
  / \                 / \
 0   2               6   7
    / \                   \
   3   4                  8
```
比如： 我们想找3跟8所在cluster的root， find(3) == 1,  find(8) == 5

```c
/*递归查找root*/
int find(int x, int parent[]) {
    int x_root = x; 
    while (parent[x_root]!= -1) {
        x_root = parent[x_root];
    }
    return x_root;
}
```
**union(x, y)**: union two cluster where x, y are in 

比如： 我们想union(3, 8),  就要先找到3的x_root， 跟8的y_root，然后合并两个root.

```c
int union(int x, int y, int parent[]) {
    int x_root = find(x, parent);
    int y_root = find(y, parent); 
    if (x_root == y_root) {
        return 0; 
    } else {
        // y_root 变成 x_root的根
        parent[x_root] = y_root;
        return 1
    }
}
```

```
              5
         /   / \
        1   6   7
       / \       \
      0   2       8
         / \ 
        3   4 
```

                                
DSU腻害的一点是优化后，用$O(1)$的average time cost, 检测图里有咩有环

两种优化： 
- Make tree flat 
- Union by rank

https://www.youtube.com/watch?v=VJnUwsE4fWA


### 分割成4个三角形， 上下左右
每个格子分割成上下左右四个三角形。
https://assets.leetcode.com/uploads/2018/12/15/3.png

每个三角形给个idx: 0, 1, 2, 3分别对应 top, right, bottom, left
```
\ 0 /
3 \ 1
/ 2 \
```
n*n的图；变成 4*n*n的数组。 初始化时数组parent的每个值都是-1, 代表每个点都是独立的。我们遍历grid中的每个点, grid[i][j]， 分别进行如下操作： 

'/':  上、左连接； 下、右连接
'\\': 上、右连接； 左、下连接
' ': 四个部分连接

对每个grid[i][j], 合并grid[i-1][j]的bottom三角形根grid[i][j]的top 三角形；合并grid[i][j-1]的right三角形跟grid[i][j]的left三角形。

最终代码如下:

```c++
class DSU {
private:
    // use array represents a graph
    vector<int> parent;
    int row = 0;
public:
    // num of root of independent cluster
    int num_root = 0;

    DSU(int n) {
        parent = vector<int>(n * n * 4, -1);
        num_root = n * n * 4;
        row = n;
    }

    int find(int x) {
        // find the root of the cluster where x is iteratively
        while (parent[x] != -1) {
            x = parent[x];
        }
        return x;
    }

    /**
    * return: 1: successfully; 0: failed
    */
    int union_cluster(int x, int y) {
        int x_root = find(x);
        int y_root = find(y);
        if (x_root == y_root) {
            return 0;
        } else {
            int min_ = min(x_root, y_root);
            int max_ = max(x_root, y_root);
            parent[min_] = max_;
            
            num_root--;
            return 1;
        };
    }

    /**
     * 将图中（i,j)位置的点，映射到数组的idx
     * @param i
     * @param j
     * @param part
     * @param n
     * @return
     */
    int idx(int i, int j, int part) {
        return (i * row + j) * 4 + part;
    }
};


// https://leetcode.com/problems/regions-cut-by-slashes/discuss/205680/JavaC%2B%2BPython-Split-4-parts-and-Union-Find
int regionsBySlashes(vector<string> &grid) {
    if (grid.empty()) return 0;
    int n = (int) grid.size();

    DSU dsu = DSU(n);

    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < n; ++j) {
            // merge the bottom part of the (i-1, j) grid and the top part of the grid (i, j)
            if (i > 0) dsu.union_cluster(dsu.idx(i - 1, j, 2), dsu.idx(i, j, 0));
            // merge the right part of the (i, j-1) grid and the left part of the grid(i,j)
            if (j > 0) dsu.union_cluster(dsu.idx(i, j - 1, 1), dsu.idx(i, j, 3));

            if (grid[i][j] == '/') {
                // union the top and the left part of this cell
                dsu.union_cluster(dsu.idx(i, j, 3), dsu.idx(i, j, 0));
                // union the right and the bottom of this cell
                dsu.union_cluster(dsu.idx(i, j, 2), dsu.idx(i, j, 1));
            }

            if (grid[i][j] == '\\') {
                dsu.union_cluster(dsu.idx(i, j, 0), dsu.idx(i, j, 1));
                dsu.union_cluster(dsu.idx(i, j, 2), dsu.idx(i, j, 3));
            }

            if (grid[i][j] == ' ') {
                dsu.union_cluster(dsu.idx(i, j, 0), dsu.idx(i, j, 2));
                dsu.union_cluster(dsu.idx(i, j, 1), dsu.idx(i, j, 3));
                dsu.union_cluster(dsu.idx(i, j, 1), dsu.idx(i, j, 2));
            }
        }
    }

    return dsu.num_root;
}

```

https://leetcode.com/problems/regions-cut-by-slashes/discuss/205680/JavaC%2B%2BPython-Split-4-parts-and-Union-Find
