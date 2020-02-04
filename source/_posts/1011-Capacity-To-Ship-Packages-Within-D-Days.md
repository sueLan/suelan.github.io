---
title: 1011. Capacity To Ship Packages Within D Days
date: 2019-06-14 15:05:10
tags:
 - Algorithm
categories: 
 - Algorithm
---
## [题目](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/)
## 分析
来自Vlad神的解答https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/discuss/256737/C%2B%2B-Binary-Search 

先定下可能的承重范围: 
1. 最大 maxCap 是所有包裹的重量和
2. 最小 minCap = max{ maxCap/D, 最重的包裹重量) 
3. 用 Binary Search 查找 `the least weight capacity of the ship`, 该船能在D天内运送完所有货物

```c++
// 求载货力为 capacity 的船运完所有货物的天数
int countDays(vector<int>& weights, int capacity) {
    int count = 1, load = 0;
    for (auto w: weights) {
        load += w;
        if (load > capacity) {
            // 超过载重能力，第二天继续运载
            load = w;
            count++;
        }
    }

    return count;
}

int shipWithinDays(vector<int>& weights, int D) {
    auto maxCap = std::accumulate(weights.begin(), weights.end(), 0);
    auto minCap = max(*max_element(weights.begin(), weights.end()), maxCap / D);
    // Binary Search 
    while (minCap < maxCap) {
        int mid = (minCap + maxCap) / 2;
        if (countDays(weights, mid) <= D) {
            maxCap = mid;
        } else {
            minCap = mid + 1;
        }
    }

    return minCap;
}

```