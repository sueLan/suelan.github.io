---
title: 792. Number of Matching Subsequences
date: 2019-07-06 11:02:58
tags:
---

## [题目](https://leetcode-cn.com/problems/number-of-matching-subsequences/)

## Solution 1 
思路：存储 + 二分查找
1. 首先将字符一集对应的下标存储在26 * n的二维数组中。比如对字符串 'abcdea'存储为

| Char | Index |
| --- | --- |
| 0 | [0, 5] |
| 1 | [1] |
| 2 | [2] |
| 3 | [3] |
| 4 | [4] |
| ... |  |
| 25 |  |

对应代码: 

```c++
  vector<vector<int>> store(26, vector<int>());
    for (int i = 0; i < S.size(); ++i) {
        // 存储字母跟index
        store[S[i] - 'a'].push_back(i);
    }
```

1. 对某个word，查找它的字符是否在二维数组中。

```c++
int numMatchingSubseq(string S, vector<string>& words) {
    vector<vector<int>> store(26, vector<int>());
    for (int i = 0; i < S.size(); ++i) {
        // 存储字母跟index
        store[S[i] - 'a'].push_back(i);
    }

    int res = 0;
    for (auto &word: words) {
        int x = -1;
        bool found = true;
        for (auto c: word) {
            // search
            auto it = upper_bound(store[c - 'a'].begin(), store[c - 'a'].end(), x);
            if (it == store[c - 'a'].end()) {
                found = false;
                break;
            } else {
                // 更新最新的下标位置
                x = *it;
            }
        }
        if (found) res++;
    }

    return res;
}
```
## Solution 2 
第二种思路，来自[Stefan大神](https://leetcode.com/problems/number-of-matching-subsequences/discuss/117634/Efficient-and-simple-go-through-words-in-parallel-with-explanation)。 
这种思路，存储的是words中等待匹配的字符的index pair。比如 words = ["a", "bb", "acd", "ace"]存储的结果是: 
|  idx | pair |
| --- | --- |
| ‘a' | [(0, 1), (2, 1), (3, 1)] |
| ‘b' | [(1, 1)] |



```c++
int numMatchingSubseq(string S, vector<string>& words) {
    vector<pair<int, int>> waiting[128];
    for (int i = 0; i < words.size(); i++)
        waiting[words[i][0]].emplace_back(i, 1);
    for (char c : S) {
        auto advance = waiting[c];
        waiting[c].clear();
        for (auto it : advance)
            waiting[words[it.first][it.second++]].push_back(it);
    }

    return waiting[0].size();
}
```

