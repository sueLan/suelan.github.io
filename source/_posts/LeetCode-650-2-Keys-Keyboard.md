---
title: 'LeetCode:650. 2 Keys Keyboard'
date: 2019-05-20 23:20:01
tags:
    - Algorithm 
categories: 
    - Algorithm 
---

## 题意: 
notepad里只有一个'A'字符串。 只能允许两个操作: 

1. Copy All: 把notepad里所有的字符串copy
2. Paste: 复制上次copy的内容

问，最少的步骤（copy & paste) 能生成n个'A'

## 分析

可以发现规律: 


| n | op | cur char |
| --- | --- | --- |
| 2 | c | A |
|  | p | AA |
| 3 | c | A |
|  | p | AA |
|  | p | AAA |
| 4 | c | A |
|  | p | AA |
|  | c | AA |
|  | p | AAAA |
| 5 | c | A |
|  | p | AA |
|  | p | AAA |
|  | p | AAAA |
|  | p | AAAAA |
| 6 | c | A |
|  | p | AA |
|  | c | AA |
|  | p | AAAAA |
|  | p | AAAAAA |

当n为质数， 所需最少步骤为n; 当n不是质数， 所需步骤为 n的质因数相加之和。比如6 = 2 x 3;它所需最小步骤 2+3 = 5

## Solution 1： 质因数分解
```c++
int minSteps(int n) {
    int ans = 0, d = 2;
    while (n > 1) {
        while (n % d == 0)  {
            ans += d;
            n /= d;
        }
        d++;
    }

    return ans;
}
```

## Solution 2: 

```c++
int minSteps(int n) {
    if (n <= 1) return 0;
    if (n <= 5) return n;
    // n>1的操作，前两步骤都是copy & paste, 从'A'变成 'AA', 所以 cur_len == 2, res == 2, 上次copy的字符数量 == 1, 如果下一次要copy & paste, 要一次copy'AA' 2个字符  
    int cur_len = 2, res = 2, next_cp_num = 2, copy_num = 1;
    while (cur_len < n) {
        if ((n - cur_len) % next_cp_num == 0) {
           // copy + paste
           cur_len += next_cp_num;
           copy_num = next_cp_num;
           // copy + paste; 2 steps
           res += 2;
        } else {
            // paste the characters copied last time 
            cur_len += copy_num;
            // only paste, 1 step
            res += 1;
        }

        // 题目要求：Copy操作要Copy所有字符
        next_cp_num = cur_len;
    }

    return res;
}
```
