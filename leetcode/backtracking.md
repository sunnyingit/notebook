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

此题的可选择项是需要画出类似于"()"，"()()", "(())"这样的组合，我们可以观察到：
1. 必须先画"("，当画了"("之后，还有两种选择：可以画"("和")"
2. 类似画"())"这样的组合不满足条件，也就说画")"的个数不能超过"("，否则不满足条件
3. 当")"画完之后，才终止

使用left和right来分别保存可以画"("和")"的次数是本题的关键，当right=0的时候终止递归调用。

```
class Solution(object):
    def generateParenthesis(self, n):
        """
        :type n: int
        :rtype: List[str]
        """
        left = right = n

        ans = []
        path = ""
        self.dfs(left, right, path, ans)

        return ans

    def dfs(self, left, right, path, ans):
        # right为0的时候，表示画完了
        if right == 0:
            ans.append(path)
            return

        # 第一个选择是可以画"("
        if left > 0:
            self.dfs(left-1, right, path + "(", ans)

        # 第二个选择是可以画")"，画")"的次数不能超过"("，也就是right不能超过left，否则不能选择画")"
        if right > left:
            self.dfs(left, right-1, path + ")", ans)
```


### 78. Subsets

```
Given an integer array nums of unique elements, return all possible subsets (the power set).
The solution set must not contain duplicate subsets. Return the solution in any order.
 
Example 1:
Input: nums = [1,2,3]
Output: [[],[1],[2],[1,2],[3],[1,3],[2,3],[1,2,3]]

Example 2:
Input: nums = [0]
Output: [[],[0]]

```

思路： 以1开头的path有：[1],[1,2],[1,2,3], 以2开头的有[2], [2,3]，以3开头的有[3]。
和46题不一样的是，选择1之后，可选项是nums[1:]， 选择2之后可选项是nums[2:], 选择3之后，可选的nums为空。

怎么去实现每次nums的变化呢，有两种方式：

```
# 使用i表示开始选择nums的范围，i每次都会+1
for i in xrange(i, len(nums)):
    self.dfs(ans, path + [nums[i]], i+1, nums)

# 直接改变nums
for i in xrange(len(nums)):
    self.dfs(ans, path + [nums[i]], nums[i+1:])
```

最终算法如下：
```
class Solution(object):
    def subsets(self, nums):
        """
        :type nums: List[int]
        :rtype: List[List[int]]
        思路： 以1开头的path有：[1],[1,2],[1,2,3], 以2开头的有[2], [2,3]，以3开头的有[3]
        """
        ans, path = [], []

        self.dfs(ans, path, 0, nums)

        return ans

    def dfs(self, ans, path, i, nums):

        ans.append(path)

        if i == len(nums):
            return

        for i in xrange(i, len(nums)):
            self.dfs(ans, path + [nums[i]], i+1, nums)


    # 第二种解法
    def dfs(self, ans, path, nums):
        ans.append(path)

        # 以1开头的时候，最后i=2时，选择的是3，此时就可以结束了
        if len(nums) == 0:
            return ans

        # 选择只能从1到2到3，然后2到3，然后3
        for j in xrange(len(nums)):
            self.dfs(ans, path+[nums[j]], nums[j+1:])
```

### 131. Palindrome Partitioning
```
Given a string s, partition s such that every substring of the partition is a palindrome.
Return all possible palindrome partitioning of s.

Example 1:

Input: s = "aab"
Output: [["a","a","b"],["aa","b"]]
Example 2:

Input: s = "a"
Output: [["a"]]
```
首先，如果不考虑组合的子字符串有[a, a, b], [aa, b], [aab], [a, ab]，首先我们需要搞清楚如何构建这些子字符串。
