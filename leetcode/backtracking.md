# BackTracking

BackTracking 是一种搜索算法，其算法公式如下：
1. Choice: 列出所有可执行的选择，依次尝试每个选择
2. ConsTraint: 处理选择的约束条件
3. Target: 完成搜索目标


### 46. Permutations
```
Given an array nums of distinct integers, return all the possible permutations. You can return the answer in any order.

Example 1:

Input: nums = [1,2,3]
Output: [[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]

Example 2:

Input: nums = [0,1]
Output: [[0,1],[1,0]]

Example 3:

Input: nums = [1]
Output: [[1]]

```

解题思路：
1. 按照backTracking的思路，首先我们列出所有的可选项，此题中，第一次搜索时，我们可以分别以1，2，3开头排列组合。
2. 约束条件是当选择了1之后，剩下的选择之后[2, 3], 同理当选择2之后，剩下的选择只有[1, 3], 选择3后，剩下的选择是[1, 2]
3. 以1开头有两种选择[2, 3]，假如第二次搜索选择2之后，剩下的只能选择3；假如第二次选择3，那最后一个选择只能是2
3. 终止条件是当nums为空时，停止搜索

```
class Solution(object):
    def permute(self, nums):
        """
        :type nums: List[int]
        :rtype: List[List[int]]
        """

        ans, path = [], []
        self.dfs(ans, path, nums)
        return ans


    def dfs(self, ans, path, nums):
        # 当nums为空时终止搜索
        if not nums:
            ans.append(path)

        # 有几种选择
        for i in range(len(nums)):
            # 下一次还有几种选择
            self.dfs(ans, path + [nums[i]], nums[:i] + nums[i+1:])
```

### 47. Permutations II

```
Given a collection of numbers, nums, that might contain duplicates, return all possible unique permutations in any order.

Example 1:

Input: nums = [1,1,2]

Output:
 [[1,1,2],
 [1,2,1],
 [2,1,1]]

Example 2:

Input: nums = [1,2,3]
Output: [[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]
```
第一题和上一题的区别是：列出可选项时，如果已经选择了，就不能再选了。
举个列子，nums=[1,1,2]，按照第一题的思路，在一轮选择中，如果择了1开头，那么本轮就不能再选择1开头，否则就重复了。
注意假如第一轮选择了1，第二轮有两个选择[1,2], 第二轮中还是可以再次选择1的。

```
class Solution(object):
    def permuteUnique(self, nums):
        """
        :type nums: List[int]
        :rtype: List[List[int]]
        """
        res = []
        # 必须先排序，基于排序来保证一轮可选项中，选择不会重复
        nums.sort()
        self.dfs(nums, [], res)
        return res


    def dfs(self, nums, path, ans):
        if not nums:
            ans.append(path)
            return

        for j in xrange(len(nums)):
            #  在一轮中选择，不能有重复的选择，有则跳过
            if j > 0  and nums[j] == nums[j - 1]:
                continue

            # 注意改变了nums的值
            self.dfs(nums[:j] + nums[j+1:], path + [nums[j]], ans
```

### 剑指Offer 38. 字符串的排列

输入一个字符串，打印出该字符串中字符的所有排列。
你可以以任意顺序返回这个字符串数组，但里面不能有重复元素。
 

```
示例:
输入：s = "abc"
输出：["abc","acb","bac","bca","cab","cba"]
```

和47一样，在一轮选择中，需要排除重复选择，47的做法是排序后，两个相邻的值如果相等，则跳过。
这道题中，使用visited字典来保存一轮中已经做出的选择，一轮选择中，只有不在visited的字母才可以被选择。

```
class Solution:
    def Permutation(self , strs):
        ans, path = [], ""

        self.dfs(strs, ans, path)

        return ans

    def dfs(self, strs, ans, path):
        if not strs:
            ans.append(path)
            return
        # 开启新一轮选择时，重新调用dfs, 会再次置空visited。
        # 需要再次强调，visited是作用于一轮选择，第二轮选择调用dfs搜索时会被重新置空。
        visited = dict()

        for i in xrange(len(strs)):
            if strs[i] in visited:
                continue
            visited[strs[i]] = 1
            self.dfs(strs[:i] + strs[i+1:], ans, path + strs[i])
```

### 22. Generate Parentheses
```
Given n pairs of parentheses, write a function to generate all combinations of well-formed parentheses.

Example 1:
Input: n = 3Output: ["((()))","(()())","(())()","()(())","()()()"]

Example 2:
Input: n = 1Output: ["()"]
```
