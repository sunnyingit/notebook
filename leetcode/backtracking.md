# BackTracking

BackTracking 是一种搜索算法，其算法公式如下：
1. Choice: 列出所有可执行的选择，依次尝试每个选择
2. ConsTraint: 处理选择的约束条件
3. Target: 完成搜索目标


BackTracking算法不能解决最多，最长，最少类似这样的问题，这种问题一般需要通过动态规划的方式解决。

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

思路：
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
思路:
1. 数组中的每个元素都可以作为数组的第一个元素，递归终止条件是nums为空
2. 多了一个约束条件，就是不能重复，举个列子，nums=[1,1,2]，按照第一题的思路，在一轮选择中，如果择了1开头，那么剩下的选择只有[2]，而不是[1, 2]。
3. 去重的方式是排序，然后比较相邻的元素，如果相等则不能再选择

```
class Solution(object):
    def permuteUnique(self, nums):
        """
        :type nums: List[int]
        :rtype: List[List[int]]
        """
        res = []
        # 排序去重
        nums.sort()

        self.dfs(nums, [], res)
        return res


    def dfs(self, nums, path, ans):
        if not nums:
            ans.append(path)
            return

        for j in xrange(len(nums)):
            #  在一轮中选择，如果相邻的值相等则跳过
            if j > 0  and nums[j] == nums[j - 1]:
                continue

            # 假设nums[1, 1, 2], 第二次调用dfs时nums变成了[1, 2]，也就是说第二次执行dfs有两种选择要么是[1]要么是[2]
            self.dfs(nums[:j] + nums[j+1:], path + [nums[j]], ans
```

### 剑指Offer 38. 字符串的排列

输入一个字符串，打印出该字符串中字符的所有排列。你可以以任意顺序返回这个字符串数组，但里面不能有重复元素。
 

```
示例:
输入：s = "abc"
输出：["abc","acb","bac","bca","cab","cba"]
```

思路：
1. 和leeetcode-47一样，在一轮选择中，需要排除重复选择，47的做法是排序后，两个相邻的值如果相等，则跳过。
2. 这道题中，使用visited字典来保存一轮中已经做出的选择，一轮选择中，只有不在visited的字母才可以被选择。

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

        # 开启新一轮选择时，再次调用dfs, 会再次置空visited
        # 有同学可能担心类似于"aabc"的字符串，最终输出是"abc"，把"a"过滤了一个，有这个担心是没有正确理解过滤的逻辑
        # 当第一次选择"a"开头后，开启第二次选择，此时visited会再次被置空，也就是说第二层选择中会再次选择"a"
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

思路：

1. 选择条件： 以1开头的path有：[1],[1,2],[1,2,3], 以2开头的有[2], [2,3]，以3开头的有[3]。
2. 和leetcode-46题不一样的是，选择1之后，可选项是nums[1:]， 选择2之后可选项是nums[2:], 选择3之后，可选的nums[3:]。

怎么去实现每次nums的变化呢，有两种方式：

```
# 使用i表示开始选择nums的范围，i每次都会+1，这样range(start, len(nums))，start的值都会加1
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
思路:
1. 如果不考虑回文，其子字符串有[a, a, b], [aa, b], [aab], [a, ab]，首先我们需要搞清楚如何构建这些子字符串。
2. 怎么判断子字符串都是回文。

第一个问题:
1. 选择条件: 构建子字符串，思路是字符串的元素个数，例如"abc"的子字符串必须是以"a"开头，长度为1的话，就是[a, 剩下的字符]，长度为2的话就是[ab, 剩下的字符串]，长度为3的话就是[abc, 剩下的字符串]
2. 约束条件: 每次构建子字符串，都需要判断是否会回文也就是s[::] == s[::-1]
3. 终止条件: 剩下的字符串为空

```
class Solution(object):
    def partition(self, s):
        """
        :type s: str
        :rtype: List[List[str]]
        思路：以a开头的可以取a[0:1]: a[0:2] a[0:3];
        当去a[0:1]的是剩下的就是a[1:]
        """
        path, ans = [], []

        self.dfs(0, ans, path, s)
        return ans

    def dfs(self, i, ans, path, s):
        if i == len(s):
            ans.append(path)
            return

        for j in xrange(i, len(s)):
            # j表示的start, 这样每次选择子字符串就是s[i:j+1]
            t = s[i:j+1]

            # 回文判断
            if t == t[::-1]:
                # 剩下的字符串就是j+1
                self.dfs(j+1, ans, path + [t], s)
```


### 93. Restore IP Addresses

```
A valid IP address consists of exactly four integers separated by single dots. Each integer is between 0 and 255 (inclusive) and cannot have leading zeros.

For example 
"0.1.2.201" and "192.168.1.1" are valid IP addresses, but "0.011.255.245", "192.168.1.312" and "192.168@1.1" are invalid IP addresses.

Given a string s containing only digits, return all possible valid IP addresses that can be formed by inserting dots into s. You are not allowed to reorder or remove any digits in s. You may return the valid IP addresses in any order.
 
Example 1:
Input: s = "25525511135"
Output: ["255.255.11.135","255.255.111.35"]

Example 2:
Input: s = "0000"
Output: ["0.0.0.0"]

Example 3:
Input: s = "101023"
Output: ["1.0.10.23","1.0.102.3","10.1.0.23","10.10.2.3","101.0.2.3"]

```
思路:
1. 选择条件: 构建子字符串，根据子字符串的长度来讲，可以选择的1,2,3....n, 不过构建的子字符串最大不超过255
2. 子字符串只能构建为4份，否则就不是合法的ip
3. 考虑01这样的子字符串的特色情况


```
class Solution(object):
    def restoreIpAddresses(self, s):
        """
        :type s: str
        :rtype: List[str]
        """
        ans = []
        self.dfs(s, 0, 0, "", ans)

        return ans



    def dfs(self, s, idx, i, path, ans):
        if idx > 4:
            return

        # 怎么判断字符串分成了四份呢，方法就是idx=4,而且此时s为空
        if idx == 4 and i == len(s):
            ans.append(path[:-1])
            return

        # 这样分为0和非0考虑
        for j in range(i, len(s)):
            # 错误的的写法是s[:i]=='0' or (s[i]!='0' and 0 < int(s[:i]) < 256)
            # 应该是要判断第一个元素不为零
            if s[i:j+1]=='0' or (s[i]!='0' and 0 < int(s[i:j+1]) < 256):
                self.dfs(s, idx+1, j+1, path+s[i:j+1]+".", ans)
```


## 79. Word Search

```
Given an m x n grid of characters board and a string word, return true if word exists in the grid.
The word can be constructed from letters of sequentially adjacent cells, where adjacent cells are horizontally or vertically neighboring. The same letter cell may not be used more than once.
 
Example 1:

Input: board = [["A","B","C","E"],["S","F","C","S"],["A","D","E","E"]], word = "ABCCED"
Output: true

```

思路:
1. 选择: 格子里面的每个点都可以作为起始的搜索点
2. 约束: 不能爬到格子外面，走过的路不能重复走
3. 如何判断找到了对应的word呢，思路是依次找到word的每个元素，直到完全找到为止

```
class Solution(object):
    def exist(self, board, word):
        """
        :type board: List[List[str]]
        :type word: str
        :rtype: bool
        """
        # path = []
        # row, col =  len(board), len(board[0])
        # for r in xrange(row):
        #     for c in xrange(col):
        #         if self.dfs(board, word, 0, r, c ,row, col, path):
        #             return True
        # return False

        path = []
        Rows, Cols = len(board), len(board[0])

        # 每个位置尝试搜索一遍
        for r in range(Rows):
            for c in range(Cols):
                # 每个位置都从word[0]开始搜索
                if self.dfsv2(board, word, 0, r, c, Rows, Cols, path):
                    return True

        return False



    def dfs(self, board, word, i, r, c, row, col, path):
        if i == len(word):
            return True


        # 不能走出格子，不能往回走，如果word[i] != board[r][c]不相等就没有必要走了
        if r < 0 or r >= row or c < 0 or c >= col or ((r, c) in path) or word[i] != board[r][c]:
            return False


        path.append((r, c))
        res = self.dfs(board, word, i+1, r+1, c, row, col, path) or \
              self.dfs(board, word, i+1, r-1, c, row, col, path) or \
              self.dfs(board, word, i+1, r, c+1, row, col, path) or \
              self.dfs(board, word, i+1, r, c-1, row, col, path)

        # 尝试后必须remove，然后重新走其他的路
        path.remove((r, c))

        return re

```


### 200. Number of Islands
```
Given an m x n 2D binary grid grid which represents a map of '1's (land) and '0's (water), return the number of islands.
An island is surrounded by water and is formed by connecting adjacent lands horizontally or vertically. You may assume all four edges of the grid are all surrounded by water.
```

思路:
1. 每个格子都可以作为搜索的起始点
2. 本题本质上就是找连在一起的"1"，上下左右都可以找，直到搜索完成，搜索完成后小岛的数量+1
3. 找到1的之后，不能再次搜索，故而设置grid[i][j] = "#"


```
class Solution(object):
    def numIslands(self, grid):
        """
        :type grid: List[List[str]]
        :rtype: int
        """

        ans=0

        # 每个格子都可以搜索
        for i in xrange(len(grid)):
            for j in xrange(len(grid[i])):
                if grid[i][j] == "1":
                    self.dfs(i, j, grid)
                    # 搜索完成后小岛数量+1
                    ans += 1

        return ans

    def dfs(self, i, j, grid):
        if i < 0 or i >= len(grid) or j < 0 or j >= len(grid[0]) or grid[i][j] != "1":
            return

        # 设置格子是非1的值
        grid[i][j] = "#"

        # 上下左右都可以查找
        self.dfs(i+1, j, grid)
        self.dfs(i-1, j, grid)
        self.dfs(i, j+1, grid)
        self.dfs(i, j-1, grid)
```

### 51. N-Queens

```
The n-queens puzzle is the problem of placing n queens on an n x n chessboard such that no two queens attack each other.
```

思路：
1. 皇后在自己所处的行，所处的列都不能放其他皇后，还有对角线也不可以
2. 正对角线：假设queen放在了(r, c), 则正其对角线positiveDiag集合是[r-c], 再次放置queen时，如果格子的r-c在集合positiveDiag中，则不能放
3. 斜对角线：假设queen放在了(r, c), 则正其对角线positiveDiag集合是[r+c], 再次放置queen时，如果格子的r+c在集合positiveDiag中，则不能放
4. 每个格子都可以试一试，注意这个遍历算法和上面的不太一样
5. 其他细节是怎么构建棋盘borad

选择行的时候，当做行放置了后，只能放置下一行，递归当r==n的时候，停止递归调用；
选择列的时候，不能处于皇后所在的列

```
class Solution:
    def solveNQueens(self , n):

        cols, positiveDiag, negtiveDiag = set(), set(), set()
        board = [["." for i in range(n)] for _ in range(n)]

        ans = []

        def dfs(r, board, ans):
            # 达到了最后一个行
            if r == n:
                board = ["".join(row) for row in board]
                ans.append(board)
                return

            # 每一行都有n列可以选择
            # 错误写法 for c in xrange(n)
            for c in xrange(n):
                # 如果当前(r,c) 处于queue所在的列， 对角线上
                if c in cols or (r - c) in positiveDiag or (r+c) in negtiveDiag:
                    continue

                # 如果不在，则放置当前位置
                board[r][c] = "Q"

                # 加入列
                cols.add(c)
                positiveDiag.add(r-c)
                negtiveDiag.add(r+c)

                dfs(r+1, board, ans)

                # remove
                cols.remove(c)
                positiveDiag.remove(r-c)
                negtiveDiag.remove(r+c)

                # 别忘了这个值重置
                board[r][c] = "."


        dfs(0, board, ans)
        return ans
```
