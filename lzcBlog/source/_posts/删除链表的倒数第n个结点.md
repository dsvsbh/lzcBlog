---
title: 删除链表的倒数第n个结点
date: 2025-02-03 18:15:21
tags: leetcode
categories: 算法
---

# 题目

[19. 删除链表的倒数第 N 个结点](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)

已解答

中等

相关标签

相关企业

提示

给你一个链表，删除链表的倒数第 `n` 个结点，并且返回链表的头结点。

**示例 1：**

![](https://assets.leetcode.com/uploads/2020/10/03/remove_ex1.jpg)

**输入：**head = [1,2,3,4,5], n = 2
**输出：**[1,2,3,5]

**示例 2：**

**输入：**head = [1], n = 1
**输出：**[]

**示例 3：**

**输入：**head = [1,2], n = 1
**输出：**[1]

**提示：**

- 链表中结点的数目为 `sz`
- `1 <= sz <= 30`
- `0 <= Node.val <= 100`
- `1 <= n <= sz`

# 题解

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        int num=0;
        ListNode p = head;
        while(p!=null)
        {
            num++;
            p=p.next;
        }
        int order = num-n+1;
        if(order<1||order>num)
        {
            return head;
        }
        if(order==1)
        {
            return head.next;
        }
        p=head;
        ListNode q=head.next;
        order--;
        while(order>0)
        {
            order--;
            if(order == 0)
            {
                p.next=q.next;
                return head;
            }
            p=p.next;
            q=q.next;
        }
        return head;
    }
}
```

# 思路

- 先扫描一次链表得到结点数

- 计算顺数的顺序

- 遍历到对应的节点，将该节点前面的节点指向该节点的后面节点

- 返回结果的头节点
