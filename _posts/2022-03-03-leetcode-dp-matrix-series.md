---
layout: post
title: "[动态规划笔记] 2 - 矩阵合集"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - algorithm
  - dp
---

leetcode 上动态规划 tag 下碰到几道矩阵的题目，记录一下解题思路。

[leetcode 221](1)  
[leetcode 1277](2)  
[leetcode 764](3) // TODO

1. leetcode 221  
  找一个 m × n 矩阵（由0和1构成）中，1 组成的正方形的最大面积。

  - 首先确定dp数组中元素的含义：同样开一个 m × n 的 dp 数组，其中 dp\[i]\[j] 的含义是以 matrix\[i]\[j]  能参与构成的最大正方形边长。单个1，构成的正方形边长就是1。
  - 递推公式：matrix\[i]\[j] = 1时，dp\[i]\[j] = min{dp\[i-1]\[j-1], dp\[i-1]\[j], dp\[i]\[j-1]} + 1。这里需要左上的 dp\[i-1]\[j-1], 否则不能判断能构成一个更大阶数的正方形。  
  ![221-image-1](/img/in-post/post-dp-matrix-series/221-img-1.png)  
  - 初始化：可以把第一行和第一列初始化为 matrix 矩阵中的值。
  - 遍历顺序：由递推公式可知我们计算元素 dp\[i]\[j] 之前需要它上一行和本行的前一个元素，因此按行从左到右计算即可  
  以下是代码：
  ```
  class Solution {
  public:
      int maximalSquare(vector<vector<char>>& matrix) {
          int ans = 0;
          vector<vector<int>> dp(matrix.size(), vector<int>(matrix[0].size(), 0));
          for(int i = 0, sz = matrix[0].size(); i < sz; i++) {
              dp[0][i] = matrix[0][i] - '0';
              ans = max(ans, dp[0][i]);
          }
          for(int i = 0, sz = matrix.size(); i < sz; i++) {
              dp[i][0] = matrix[i][0] - '0';
              ans = max(ans, dp[i][0]);
          }
          
          for(int i = 1, m = matrix.size(); i < m; i++) {
              for(int j = 1, n = matrix[0].size(); j < n; j++) {
                  if(matrix[i][j] == '1') {
                      dp[i][j] = min({dp[i-1][j-1], dp[i-1][j], dp[i][j-1]}) + 1;
                      ans = max(ans, dp[i][j]);
                  }
              }
          }
          return ans * ans;
      }
  };
  ```

2. leetcode 1277  
  这题变成了数矩阵中有多少个正方形，比如下图中有10个边长为1，4个边长为2，1个边长为3，总计15个正方形。  
  ![1277-img-1](/img/in-post/post-dp-matrix-series/1277-img-1.png)  
  还是在leetcode discuss看到的，这个题依然可以用前面221题的解法做，此时dp数组元素 dp\[i]\[j] 的含义变为 matrix\[i]\[j] 元素参与构成的正方形数目。然后对 dp 数组求和就得到最后结果，就不走dp解题的套路了贴一下代码吧。  
  ```
  class Solution {
  public:
      int countSquares(vector<vector<int>>& matrix) {
          int m = matrix.size(), n = matrix[0].size();
          vector<vector<int>> dp(m, vector<int>(n, 0));
          dp[0] = matrix[0];
          for(int i = 0; i < m; i++){
              dp[i][0] = matrix[i][0];
          }
          
          int sum = accumulate(dp[0].begin(), dp[0].end(), 0);
          for(int i = 1; i < m; i++){
              sum += dp[i][0];
              for(int j = 1; j < n; j++){
                  if(matrix[i][j] == 1) {
                      dp[i][j] = min({dp[i - 1][j], dp[i][j - 1], dp[i-1][j-1]}) + 1;
                  }
                  sum += dp[i][j];
              }
          }
          return sum;
      }
  };
  ```

3. leetcode 764 // TODO  


--------


[1]: https://leetcode.com/problems/maximal-square/
[2]: https://leetcode.com/problems/count-square-submatrices-with-all-ones/
[3]: https://leetcode.com/problems/largest-plus-sign/solution/


