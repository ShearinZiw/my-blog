+++
date = '2026-04-01T15:00:01+08:00'
title = 'Leetcode'
+++
# 动态规划

## [[309]买卖股票的最佳时机含冷冻期](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-with-cooldown/description)

给定一个整数数组prices，其中第  prices[i] 表示第 i 天的股票价格 。​

设计一个算法计算出最大利润。在满足以下约束条件下，你可以尽可能地完成更多的交易（多次买卖一支股票）:

卖出股票后，你无法在第二天买入股票 (即冷冻期为 1 天)。
注意：你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票）。

- hold(持有)
  - keep hold
    - 保持持有
  - selled
    - 把持有的买了
      - price + hold  
- selled(不持有，在冷静期)
  - sell
    - 出冷静期
- sell(不持有，可以卖)
  - keep sell
    - 继续不持有
  - hold
    - 买入
      - sell - price

```cpp
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int hold = -prices[0];
        int sell = 0;
        int selled = 0;

        for(vector<int>::iterator it = prices.begin()+1; it != prices.end(); ++it){
            int tmp = hold + *it;
            hold = max(hold,sell-*it);
            sell = max(sell,selled);
            selled = tmp;
        }

        return max(sell,selled);
    }
};
```

# 回溯



## [17.电话号码的字母组合](https://leetcode.cn/problems/letter-combinations-of-a-phone-number/description)

