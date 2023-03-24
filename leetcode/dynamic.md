# Dynamic Programming

动态规划的思想是将一个问题拆成几个子问题，分别求解这些子问题，即可推断出大问题的解。解题步骤:
1. 子问题的拆分和定义，也就是状态的定义
2. 寻找子问题的初始状态
3. 解决子问题如何推导出大问题，也就是列出状态的转移方程


动态规划一般是用来解决有没有解，最优解等问题，比如最多，最少等问题，相对于回溯算法暴力搜索，动态规划会更优雅。


## 300. Longest Increasing Subsequence

```
Given an integer array nums, return the length of the longest strictly increasing subsequence.
A subsequence is a sequence that can be derived from an array by deleting some or no elements without changing the order of the remaining elements. For example, [3,6,2,7] is a subsequence of the array [0,3,1,6,2,2,7].
 
Example 1:
Input: nums = [10,9,2,5,3,7,101,18]
Output: 4
Explanation: The longest increasing subsequence is [2,3,7,101], therefore the length is 4.

```

思路:
1. 我们需要求解最长的增序子数组，我们设置dp[i]表示从num[0]到num[i]的增序个数
2. 状态转移: 如何判断dp[i]的值呢，错误思路是dp[i] = dp[i] if nums[i] <= nums[i-1] or d[i-1] + 1 if nums[i] > nums[i-1]，只比较nums[i]和nums[i-1]是不对的
3. 正确的 dp[i] = max(dp[n]+1, dp[i]) for 0-n if dp[n] < dp[i]

```
class Solution(object):
    def lengthOfLIS(self, nums):
        """
        :type nums: List[int]
        :rtype: int
        """

        # 初始状态设定
        dp = [1 for _ in xrange(len(nums))]

        for i in xrange(1, len(nums)):
            # 遍历nums[:i]，找到dp[i]的值
            for j in xrange(i):
                if nums[j] < nums[i]:
                    dp[i] =  max(dp[i], dp[j]+1)

        # 返回最大的值
        return max(dp)
```


### 53. Maximum Subarray
```
53. Maximum Subarray
Given an integer array nums, find the contiguous subarray (containing at least one number) which has the largest sum and return its sum.
A subarray is a contiguous part of an array.
 
Example 1:
Input: nums = [-2,1,-3,4,-1,2,1,-5,4]
Output: 6
Explanation: [4,-1,2,1] has the largest sum = 6.
```

思路:
1. 设定dp[i] 表示从0到i数字之和
2. dp[i] = dp[i-1] + nums[i] if dp[i-1] > 0，也就是说nums[i]前面的数字累加之和大于0才累加，否则不用累加

```
class Solution(object):
    def maxSubArray(self, nums):
        """
        :type nums: List[int]
        :rtype: int
        """
        if not nums:
            return 0

        # 初始状态设置
        dp = [0 for _ in xrange(len(nums))]

        # 第一个元素的之和就是nums[0]
        dp[0] = nums[0]

        for i in xrange(1, len(nums)):
            if dp[i-1] > 0:
                dp[i] = dp[i-1] + nums[i]
            else:
                dp[i] = nums[i]


        return max(dp)
```

### 70. Climbing Stairs

you are climbing a staircase. It takes n steps to reach the top.
Each time you can either climb 1 or 2 steps. In how many distinct ways can you climb to the top?

思路：
1. dp[i] 表示从0到i有多少种走法
2. 到第i台阶方法可以从i-1台阶走一步，也可以从i-2台阶走两步到台阶i， 所以dp[i] = dp[i-1] + dp[i-2]

```
class Solution(object):
    def climbStairs(self, n):
        """
        :type n: int
        :rtype: int
        """

        if n <= 1:
            return n

        dp = [0 for _ in range(n)]
        dp[0] = 1
        dp[1] = 2


        if n <= 2:
            return dp[n-1]


        for i in xrange(2, n):
            dp[i] = dp[i-1] + dp[i-2]

        return dp[n-1]

```


### 746. Min Cost Climbing Stairs

```
You are given an integer array cost where cost[i] is the cost of ith step on a staircase. Once you pay the cost, you can either climb one or two steps.

You can either start from the step with index 0, or the step with index 1.

Return the minimum cost to reach the top of the floor.

Example 1:

Input: cost = [10,15,20]
Output: 15
Explanation: You will start at index 1.
- Pay 15 and climb two steps to reach the top.
The total cost is 15.
```

本题求解到达顶点需要的最少的cost, 思路：
1. 主要思路是从计算好从倒数第一个，或者第二个位置走到顶端需要最短的距离, 然后在计算从后面的位置走到倒数第二个或者倒数第一个最多的距离
1. 到底顶点有两种走法，由cost[i]走一步到顶，或者cost[i-1]走两步，到达顶点的最小dp[i] = min(cost[i+1], cost[i+2])
2.

```
class Solution(object):
    def minCostClimbingStairs(self, cost):
        """
        :type cost: List[int]
        :rtype: int
        """

        if not cost:
            return 0
        # [2, 5, 10, 0] 达到0才是顶端，达到0有两种走法，一步或者两步
        # 可以从5走两步到0，也可以从20走一步到0，开销分别是5和20
        # 从i到下一个节点有两种走法：
        # cost[i] = cost[i] + min(cost[i+1], cost[i+2])
        cost.append(0)

        for i in range(len(cost) - 3, -1, -1):
            # 从向前走两步的消耗是到达cost[i]的费用加上min(cost[i+1], cost[i+2])
            cost[i] += min(cost[i+1], cost[i+2])


        return min(cost[0], cost[1])
```
