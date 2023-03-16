# 链表
链表的题型主要有三大类:
1. 链表是否有环。
2. 链表节点删除。
3. 链表节点翻转。


## 链表有环判断

### 141. Linked List Cycle
```
Given head, the head of a linked list, determine if the linked list has a cycle in it.

There is a cycle in a linked list if there is some node in the list that can be reached again by continuously following the next pointer. Internally, pos is used to denote the index of the node that tail's next pointer is connected to. Note that pos is not passed as a parameter.

Return true if there is a cycle in the linked list. Otherwise, return false.
```

思路: 快慢指针，在一个圆圈里面跑，快慢指针一定会相遇。

```
class Solution(object):
    def hasCycle(self, head):
        """
        :type head: ListNode
        :rtype: bool
        """
        fast = slow = head

        while fast and fast.next:
            fast = fast.next.next
            slow = slow.next

            if slow == fast:
                return True
        return False
```


### 142. Linked List Cycle II

```
Given the head of a linked list, return the node where the cycle begins. If there is no cycle, return null.

There is a cycle in a linked list if there is some node in the list that can be reached again by continuously following the next pointer. Internally, pos is used to denote the index of the node that tail's next pointer is connected to (0-indexed). It is -1 if there is no cycle. Note that pos is not passed as a parameter.

Do not modify the linked list.
```

思路:
1. 找到到快慢指针相遇的节点
2. 分别从head到相遇的节点开始走，他们再次相遇的地方就是环的起点。
3. 这是个数学题，参考视频：https://www.youtube.com/watch?v=UkKBPGt5Nok


```
class Solution(object):
    def detectCycle(self, head):
        """
        :type head: ListNode
        :rtype: ListNode

        """

        fast = slow = head

        while fast and fast.next:
            fast = fast.next.next
            slow = slow.next

            if slow == fast:
                break

        # 注意
        if not fast or not fast.next:
            return

        # 再次相遇的点就是环的启动
        p = head
        while p != slow:
            p = p.next
            slow = slow.next
        return
```

## 链接节点删除

### 19. Remove Nth Node From End of List

```
Given the head of a linked list, remove the nth node from the end of the list and return its head.
Input: head = [1,2,3,4,5], n = 2
Output: [1,2,3,5]
```
思路：
1. 本题的关键是找到node(4)，通常能想到的方法是计算list的长度, 第二种方法是快慢指针，快指针先走N步，然后快慢指针一起走，直到快指针走完。
2. 特殊情况是n等于list的长度，返回的是head.next的值

```
class Solution(object):
    def removeNthFromEnd(self, head, n):
        """
        :type head: ListNode
        :type n: int
        :rtype: ListNode

        """

        fast = slow = head

        while n > 0:
            fast = fast.next
            n -= 1

        if not fast:
            return head.next

        # 这里条件判断容易错误的写成while fast:
        while fast.next:
            fast = fast.next
            slow = slow.next

        slow.next = slow.next.next
        return head
```


### 82. Remove Duplicates from Sorted List II
```
Given the head of a sorted linked list, delete all nodes that have duplicate numbers, leaving only distinct numbers from the original list. Return the linked list sorted as well.

Input: head = [1,2,3,3,4,4,5]
Output: [1,2,5]

Input: head = [1,1,1,2,3]
Output: [2,3]
```
思路：
1. 可能会删除第一个节点，所以必须设置dummy和pre指向head
2. 比较相邻的node的值，直到不相等为止
3. 比如list=-1->1， pre=Node(-1)则不需要删除，必须pre.next.next存在时才可能会删除，所以遍历终止条件是：pre and pre.next and pre.next.next都不为空

```
class Solution(object):
    def deleteDuplicates(self, head):
        """
        :type head: ListNode
        :rtype: ListNod
        1. 对比cur.val和next.val的值，如果相等，需要通过while一直比较直到不相等为止。
        2. 结束条件 pre and pre.next and pre.next.next
        """
        dummy = pre = ListNode(-1, head)

        # -1->1 只有一个元素就不用再删除了
        while pre and pre.next and pre.next.next:
            cur = pre.next
            n = cur.next

            # -1->1->1->1->1->2 需要考虑有多个1的情况下删除
            # 必须判断这种情况
            if cur.val == n.val:
                while n and n.val == cur.val:
                    n = n.next
                pre.next = n
            else:
                pre = pre.next

        return dummy.next
```

### 83. Remove Duplicates from Sorted List
```
Given the head of a sorted linked list, delete all duplicates such that each element appears only once. Return the linked list sorted as well.

Input: head = [1,1,2]
Output: [1,2]

```
思路：依次比较相邻的两个节点即可。

```
class Solution(object):
    def deleteDuplicates(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        cur = head

        while cur and cur.next:
            n = cur.next

            if cur.val == n.val:
                cur.next = n.next
            else:
                cur = cur.next

        return hea
```


## 链表节点翻转

### 206. Reverse Linked List

```
Given the head of a singly linked list, reverse the list, and return the reversed list.
```

思路：遍历list修改每个node的next指向，遇到需要改变node指向的题目，大部分情况都需要设置一个newHead变量。

```
class Solution:
    def ReverseList(self, head):
        newHead = None

        while head:
            n = head.next
            head.next = newHead

            # 赋值顺序不能乱，千万不能写成head=n在执行newHead=head
            newHead = head
            head = n

        return newHead
```

### 92. Reverse Linked List II

```
Given the head of a singly linked list and two integers left and right where 
left <= right, reverse the nodes of the list from position left to position right, and return the reversed list.

Example 1:

Input: head = [1,2,3,4,5], left = 2, right = 4

Output: [1,4,3,2,5]
```

以1->2->3->4->5，left=2, right=4为例，把2->3->4翻转为4->3->2 然后连接为1->(4->3->2)->5，需要设置几个核心变量：

1. 需要设置一个变量保存node(2)前面的节点pre=node(1)
2. 需要设置一个变量rightNode=node(4), 设置一个变量endNode指向rightNod.next, 也就是endNode=node(5)
3. 从pre.next开始翻转，直到node(5)停止翻转
4. 翻转完成后，pre.next指向node(4), node(2)指向endNode(5)


```
class Solution(object):
    def reverseBetween(self, head, left, right):
        """
        :type head: ListNode
        :type left: int
        :type right: int
        :rtype: ListNode
        """

        dummy = pre = ListNode(-1)
        dummy.next = head

        while left > 1 and pre:
            left -= 1
            pre = pre.next

        rightNode = head
        while right > 1 and rightNode:
            right -= 1
            rightNode = rightNode.next

        endNode = rightNode.next


        # reverse cur初始化是1，在cur等于5的时候停止
        cur = pre.next
        newHead = None
        while cur != endNode:
            n = cur.next
            cur.next = newHead

            newHead = cur
            cur = n

        # 本题pre理解为1， tail理解为2
        tail = pre.next
        # rightNode理解为4
        pre.next = rightNode
        # endNode理解为5
        tail.next = endNode

        return dummy.nex

```

### 25. Reverse Nodes in k-Group

```
Given the head of a linked list, reverse the nodes of the list k at a time, and return the modified list.
k is a positive integer and is less than or equal to the length of the linked list. If the number of nodes is not a multiple of k then left-out nodes, in the end, should remain as it is.
You may not alter the values in the list's nodes, only nodes themselves may be changed.
 
Example 1:
Input: head = [1,2,3,4,5], k = 2
Output: [2,1,4,3,5]

Example 2:
Input: head = [1,2,3,4,5], k = 3
Output: [3,2,1,4,5]
```

可以参考leetcode-92题的解题思路，设置groupPre，rightNode， endNode变量分别指向开始翻转的前一个节点，翻转的最后一个节点，翻转停止的节点。
和leetcode-92的思路一样，我们先找到rightNode，也就是最后需要翻转的节点，如果找不到，说明整个链表的节点数小于K，也就是翻转结束。
本题的关键是写一个辅助函数getRightNode用于查找rightNode节点。

```
# Definition for singly-linked list.
# class ListNode(object):
#     def __init__(self, val=0, next=None):
#         self.val = val
#         self.next = next


class Solution(object):
    def reverseKGroup(self, head, k):
        """
        :type head: ListNode
        :type k: int
        :rtype: ListNode


        """
        groupPre = dummy = ListNode(-1, head)

        while True:
            rightNode = self.getRightNode(groupPre, k)
            if not rightNode:
                break

            cur = groupPre.next
            newHead = None

            # reverse直到cur = groupEnd
            endNode = rightNode.next

            while cur != endNode:
                n = cur.next
                cur.next = newHead

                newHead = cur
                cur = n

            tail = groupPre.next
            groupPre.next = rightNode
            tail.next = endNode

            # 开始第二次翻转，比如-1->1->2->3->4;k=2;第一次翻转后-1>2->1->3->4;
            # 下一次需要从node(3)开始翻转，那node(3)的pre就是node(1)，也就是tail指向的节点
            groupPre = tail

        return dummy.next

    def getRightNode(self, cur, k):
        while cur and k > 0:
            cur = cur.next
            k -= 1
        return cur

```
## 链表节点交换

### 24.Swap Nodes in Pairs

```
Given a linked list, swap every two adjacent nodes and return its head. You must solve the problem without modifying the values in the list's nodes (i.e., only nodes themselves may be changed.)

example 1:
Input: head = 1->2->3->4
Output: 2->1->4->3
```

以1->2->3->4为例，需要修改头节点的指针，所以通常需要设置一个dummy指向head，这样设置之后就变成了-1->1->2->3->4。
1. 需要知道-1->1不需要swap，只有-1->1->2才需要swap，这是遍历终止的条件
2. 设置几个变量保存关键节点: pre=Node(-1)，cur = pre.next=Node(1); n=cur.next=Node(2), nn=n.next=Node(3)
3. swap就是让-1->2->1->3->4，也就是pre.next=n; n.next=cur; cur.next=nn;
4. 第二轮swap的pre是Node(1)

本题只需要记住`pre，cur, n, nn`这几个关键节点思路就顺畅了。

```
class Solution(object):
    def swapPairs(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        pre = dummy = ListNode(-1)
        pre.next = head

        # -1->1不需要swap，只有-1->1->2才需要swap
        while pre and pre.next and pre.next.next:
            # 用几个变量指向关键节点
            cur = pre.next
            n = cur.next
            nn = n.next

            # 改变节点指向
            pre.next = n
            n.next = cur
            cur.next = nn

            # 第二轮swap的pre变成Node(1), 注意不能写成pre = pre.next因为在前面已经修改了pre.next的指向，只能等于cur节点
            pre = cur

        return dummy.nex
```

### 61. Rotate List

```
Given the head of a linked list, rotate the list to the right by k places.
 
Example 1:

Input: head = [1,2,3,4,5], k = 2
Output: [4,5,1,2,3]
```

思路：先把收尾连接起来node(5)->node(1) 然后找到node(3)，设置node(3).next=None即可。
那问题就变成了怎么实现收尾相连，怎么知道Node(3)?

最重要的是，收尾相连之前，需要判断`k%len(list)==0`是否等于0，如果等于0就直接返回。
这个判断一定要在连接链表首尾之前做，否则会构成循环链表。

```
# Definition for singly-linked list.
# class ListNode(object):
#     def __init__(self, val=0, next=None):
#         self.val = val
#         self.next = next
class Solution(object):
    def rotateRight(self, head, k):
        """
        :type head: ListNode
        :type k: int
        :rtype: ListNode
        """
        if not head or not head.next:
            return head


        # 找到tail节点
        l = 1
        tail = head
        while tail.next:
            tail = tail.next
            l += 1

        # 千万别忘了k%l的运行
        k = (k % l)
        if k == 0:
            return head

        # p指向最后一个节点
        tail.next = head

        # 找到Node(3)
        move = l - k
        while move > 1 and head:
            head = head.next
            move -= 1

        newTail = head
        newHead = newTail.next
        newTail.next = None

        return newHead

```

### 143. Reorder List

```
You are given the head of a singly linked-list. The list can be represented as:

L0 → L1 → … → Ln - 1 → Ln
Reorder the list to be on the following form:

L0 → Ln → L1 → Ln - 1 → L2 → Ln - 2 → …
You may not modify the values in the list's nodes. Only nodes themselves may be changed.

Example 1:
Input: head = [1,2,3,4]
Output: [1,4,2,3]
```

如果用leetcode-24解题思路直接swap，基本上不可能做出来此题，本题的正确思路分为三步:
1. 第一步找到Middle Node，也就是node(2)
2. 翻转node(3)->node(4)变成node(4)->node(3)
3. 合并node(1)->node(2)和node(4)->node(3)两个列表，合并过程如下：

```
class Solution(object):
    def reorderList(self, head):
        """
        :type head: ListNode
        :rtype: None Do not return anything, modify head in-place instead.
        """
        # 找到Node(2)
        middle = self.findMiddle(head)

        # 翻转3->4变成4->3
        l2 = self.reverse(middle.next)

        # 断开Node(2)的指向
        middle.next = None

        # 合并
        return self.merge(head, l2)


    # 1->2->3->4 则slow返回的node(3)
    def findMiddle(self, head):
        fast = slow = head
        while fast and fast.next:
            fast = fast.next.next
            slow = slow.next

        return slow


    def reverse(self, head):
        newHead = None

        while head:
            n = head.next
            head.next = newHead
            newHead = head
            head = n

        return newHead

    # l1 = 1->2->3  l2 = 5->4
    def merge(self, l1, l2):
        dummy = ListNode(-1)
        dummy.next = l1

        while l1 and l2:
            # 需要先保存后l1.Next，因为后续会改变l1.next的指向
            l1Next = l1.next
            l1.next = l2

            # 需要先保存后l2.Next，因为后续会改变l1.next的指向
            l2Next = l2.next
            l2.next = l1Next

            l1 = l1Next
            l2 = l2Next
        return dummy.next
```




### 21. Merge Two Sorted Lists
```
You are given the heads of two sorted linked lists list1 and list2.
Merge the two lists in a one sorted list. The list should be made by splicing together the nodes of the first two lists.
Return the head of the merged linked list.
 
Example 1:
Input: list1 = [1,2,4], list2 = [1,3,4]
Output: [1,1,2,3,4,4]
```
思路：设置dummy和pre节点，通过比list1, list2节点的大小，依次连接即可。

```
class Solution(object):
    def mergeTwoLists(self, list1, list2):
        """
        :type list1: Optional[ListNode]
        :type list2: Optional[ListNode]
        :rtype: Optional[ListNode]
        """

        dummy = pre = ListNode(-1)

        while list1 and list2:
            if list1.val <= list2.val:
                pre.next = list1
                list1 = list1.next
            else:
                pre.next = list2
                list2 = list2.next
            pre = p.next

        if list1:
            pre.next = list1

        if list2:
            pre.next = list2

        return dummy.next

```
### 2. Add Two Numbers

```
You are given two non-empty linked lists representing two non-negative integers. The digits are stored in reverse order, and each of their nodes contains a single digit. Add the two numbers and return the sum as a linked list.
You may assume the two numbers do not contain any leading zero, except the number 0 itself.
 
Example 1:

Input: l1 = [2,4,3], l2 = [5,6,4]
Output: [7,0,8]
```

思路：
1. 还是那句话，只要需要构建新的list，都需要设置dummy和pre节点，然后依次计算新node的value
2. 设置carry保存上一次相加的进位值

```
class Solution(object):
    def addTwoNumbers(self, l1, l2):
        """
        :type l1: ListNode
        :type l2: ListNode
        :rtype: ListNode
        """
        dummy = pre = ListNode(-1)
        carry = 0

        while l1 or l2 or carry:
            total = 0
            if l1:
                total += l1.val
                l1 = l1.next
            if l2:
                total += l2.val
                l2 = l2.next

            total += carry


            carry = total // 10
            val = total % 10

            node = ListNode(val)
            pre.next = node
            pre = node

        return dummy.next
```

### 328. Odd Even Linked List
```
Given the head of a singly linked list, group all the nodes with odd indices together followed by the nodes with even indices, and return the reordered list.
The first node is considered odd, and the second node is even, and so on.
Note that the relative order inside both the even and odd groups should remain as it was in the input.
You must solve the problem in O(1) extra space complexity and O(n) time complexity.

Input: head = [1,2,3,4,5]
Output: [1,3,5,2,4]

```
思路：设置两个oddHead和evenHead, 最终遍历完成后，oddHead=1->3->5，evenHead=2->4->5，所以一定要设置preEven.next=None，否则会成环。


```
class Solution(object):
    def oddEvenList(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """

        oddHead = preOdd = ListNode(-1)
        evenHead = preEven = ListNode(-1)

        k = 1
        while head:
            if k % 2 == 0:
                preEven.next = head
                preEven = head
            else:
                preOdd.next = head
                preOdd = head
            k += 1
            head = head.next

        # 设置node(4)的next为空
        preEven.next = None
        preOdd.next = evenHead.next

        return oddHead.next

```

###  86. Partition List

```
Given the head of a linked list and a value x, partition it such that all nodes less than x come before nodes greater than or equal to x.
You should preserve the original relative order of the nodes in each of the two partitions.

Input: head = [1,4,3,2,5,2], x = 3
Output: [1,2,2,4,3,5]
```
思路：和leetcode-328题一样，设置两个变量分别保存大于X和小于X的链表


```
class Solution(object):
    def partition(self, head, x):
        """
        :type head: ListNode
        :type x: int
        :rtype: ListNode
        """

        lessHead = less = ListNode(-1)
        largeHead = large = ListNode(-1)

        while head:
            if head.val < x:
                less.next = head
                less = head
            else:
                large.next = head
                large = head

            head = head.next

        # 必须把large.next断开，否则可能会构建成环
        large.next = None
        less.next = largeHead.next

        return lessHead.next
```

### 234. Palindrome Linked List
```
Given the head of a singly linked list, return true if it is a palindrome.

Example 1:
Input: head = [1,2,2,1]
Output: true

Example 2:
Input: head = [1,2]
Output: false
```
思路：
1. 找到中间节点，例如2->1->3->1->2，找到节点node(2)
2. reverse中间节点2->1变为1->2
3. 比较两部分head=1->2->3和reverse=1->2, 如果reverser的节点都和head相同，则表示Palindrome。

```
class Solution(object):
    def isPalindrome(self, head):
        """
        :type head: ListNode
        :rtype: bool
        """

        if not head or not head.next:
            return True

        mid = self.findMid(head)

        reverseHead = self.reverse(mid)

        # 1->2->3->2->1 middle是3， middle.next = 2->1 reverse之后1->1
        # 而 head = 1->2->3->2->1 所以只要以 reverse存在
        while reverseHead:
            if head.val != reverseHead.val:
                return False

            head = head.next
            reverseHead = reverseHead.next

        return True

    def findMid(self, head):
        fast = slow = head

        while fast and fast.next:
            fast = fast.next.next
            slow = slow.next

        return slow

    def reverse(self, head):
        newHead = None
        while head:
            next = head.next
            head.next = newHead

            newhead = head
            head = next
```


### 138. Copy List with Random Pointer
```
A linked list of length n is given such that each node contains an additional random pointer, which could point to any node in the list, or null.
Construct a deep copy of the list. The deep copy should consist of exactly n brand new nodes, where each new node has its value set to the value of its corresponding original node. Both the next and random pointer of the new nodes should point to new nodes in the copied list such that the pointers in the original list and copied list represent the same list state. None of the pointers in the new list should point to nodes in the original list.
For example, if there are two nodes X and Y in the original list, where X.random --> Y, then for the corresponding two nodes x and y in the copied list, x.random --> y.
Return the head of the copied linked list.
The linked list is represented in the input/output as a list of n nodes. Each node is represented as a pair of [val, random_index] where:
val: an integer representing 
Node.val
random_index: the index of the node (range from 
0 to 
n-1) that the 
random pointer points to, or 
null if it does not point to any node.
Your code will only be given the head of the original linked list.
 
Example 1:

Input: head = [[7,null],[13,0],[11,4],[10,2],[1,0]]
Output: [[7,null],[13,0],[11,4],[10,2],[1,0]]

Example 2:

Input: head = [[1,1],[2,1]]
Output: [[1,1],[2,1]]

Example 3:

Input: head = [[3,null],[3,0],[3,null]]
Output: [[3,null],[3,0],[3,null]]
```

思路：
1. 深度拷贝需要重新构建一个newNode,  遍历head使用一个dic保存oldNode到newNode的映射；
2. 再次遍历head，通过oldNode就可以找到newNode, 而且通过oldNode.next可以知道newNode，因为都保存在dic中


```
class Solution(object):
    def copyRandomList(self, head):
        """
        :type head: Node
        :rtype: Node
        """
        if not head:
            return head

        m = n = head

        dic = dict()
        # 使用dict保存, key为旧的node，value为新的node
        while m:
            newNode = Node(m.val)
            dic[m] = newNode
            m =  m.next

        while n:
            newNode = dic[n]

            # next指向下一个node的next, 最后一个next为空，需要判断
            newNode.next = dic.get(n.next, None)

            # randome执行当前节点的random的拷贝
            newNode.random = dic.get(n.random, None)
            n = n.next

        return dic[head]
```
