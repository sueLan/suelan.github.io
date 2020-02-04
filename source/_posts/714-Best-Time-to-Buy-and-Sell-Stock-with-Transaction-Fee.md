---
title: 714. Best Time to Buy and Sell Stock with Transaction Fee
date: 2019-06-06 10:59:46
tags:
 - Algorithm 
categories:
 - Algorithm 
---

##  [题目](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-transaction-fee)

## 分析

### 定义两个状态变量: 
- buy_profit: 若i-1天，持有股票，最大利润是 $(buy_profit)
- sell_profit: 若i-1天，卖出股票, 最大利润是 $(sell_profit)

### 买卖利润情况： 

**最开始**: i == 0;  `int buy_profit = -prices[0], sell_profit = 0`
**第i天**: 

- 当天买: 
    - 维持i-1天持有, 不变； $buy\_profit_i == buy\_profit_{i-1}$ 
    - i-1天卖出，i天买进 (因为题目要求每次只能买1 share of stock, 买入前必须把手上的stock都卖出); $buy\_profit_i = sell\_profit_{i-1} - prices[i]$
- 当天卖:  
    - i-1天持有，i天卖出; 当天的利润为 $sell\_profit_i=buy\_profit_i-1 + prices[i] - fee$
    - i-1天已经卖出，当天不操作；利润为 $sell\_profit_i == sell\_profit_{i-1}$ 
    
每个i迭代，我们都要考虑这四种情况，计算出buy_profit跟sell_profit的局部最优. 


### 例子
Input: prices = [1, 3, 2, 8, 4, 9], fee = 2

最优操作： 

|  | 1 | 3 | 2 | 8 | 4 | 9 |
| --- | --- | --- | --- | --- | --- | --- |
| max buy_profit | -1 | -1 max{-1, -3} | -1 max{-1, -2}| -1 max{-1, 8}| 1 max{1, -4} | 1 max{1, -4}|
| max sell_profit | 0 | 0 max{0, 0}| 0 max{0, -1}| 5 max{0, 5}| 5 max{5, 3}| 8 {5, 8}|

### 贪心算法
A greedy algorithm always makes the choice that `looks best at the moment` That is, it makes a `locally optimal choice` in the hope that this choice will `lead to a globally optimal solution`. 


## Greedy Algorithm   


```c++
int maxProfit(vector<int>& prices, int fee) {
    if (prices.empty()) return 0;
    int buy_profit = -prices[0], sell_profit = 0;
    for (int i = 1; i < prices.size(); ++i) {
        buy_profit = max(buy_profit, sell_profit - prices[i]);
        sell_profit = max(sell_profit, buy_profit + prices[i] - fee);
     }

    return sell_profit;
}
```
或者

```c++
    int maxProfit(vector<int>& prices, int fee) {
      int n = prices.size();
      if (n <= 1 ) return 0; // need at least 2 days to make a transactions
      
      vector<int> buys (n); // buys [i] means max money on day i ending in buy  state (have stock in hand)
      vector<int> sells(n); // sells[i] means max money on day i ending in sell state (no stock in hand)
      
      buys [0] = -prices[0];
      sells[0] = 0;
      
      for (int i = 1; i < n; ++i) {
        buys [i] = max(buys [i-1], sells[i-1]-prices[i]);     // buy  state to buy  state: continue holding onto stock
                                                              // sell state to buy  state: buy on day i
        sells[i] = max(sells[i-1], buys [i-1]+prices[i]-fee); // sell state to sell state: continue not holding any stock
                                                              // buy  state to sell state: sell on day i
      }
      return sells[n-1];
    }
```
时间复杂度: $o(n)$