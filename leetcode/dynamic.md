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
