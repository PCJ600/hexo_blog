---
layout: next
title: leetcode几道链表题解决思路
date: 2021-02-06 19:43:52
categories: LeetCode
tags: LeetCode
---

### 前言

LeetCode链表题汇总，记录解题思路，C/C++语言实现。

### LeetCode链表题

* 判断链表是否有环 ：[https://leetcode-cn.com/problems/linked-list-cycle/](https://leetcode-cn.com/problems/linked-list-cycle/)

* 输出环形链表的入环点：[https://leetcode-cn.com/problems/linked-list-cycle-ii/](https://leetcode-cn.com/problems/linked-list-cycle-ii/)

* 输出链表中倒数第k个节点：[https://leetcode-cn.com/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/](https://leetcode-cn.com/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/)

* 反转链表：[https://leetcode-cn.com/problems/reverse-linked-list/](https://leetcode-cn.com/problems/reverse-linked-list/)

<!-- more -->

### 解题思路

#### 判断链表是否有环

思路：快慢指针法。 慢指针每前进一步，快指针就前进两步。如果快慢指针能相遇，证明有环，反之无环。

```C++
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
bool hasCycle(ListNode *head) {
    ListNode *fast = head;
    ListNode *slow = head;
    while (fast != NULL && fast->next != NULL) {
        fast = fast->next->next;
        slow = slow->next;
        if (fast == slow) { // 如果快慢指针能相遇，说明有环。
            return true;
        }
    }
    return false;
}
```



#### 输出环形链表的入环点

以下给出两种思路：

**方法一：** 遍历一次链表，将遍历到的节点存储到哈希表，如果节点已经在哈希表中，说明有环，输出入环点。

**复杂度分析：** 时间复杂度O(N),  空间复杂度O(N)

```C++
ListNode *detectCycle(ListNode *head) {
    std::unordered_map<ListNode *, int> t; // 用哈希表将遍历过节点保存下来
    while(head) {
        if (t.find(head) != t.end()) {
            return head;
        }
        t[head] = 1;
        head = head->next;
    }
    return NULL;
}
```

**方法二：** 快慢指针法。创建fast, slow两个指针指向链表头，令fast前进速度为slow的两倍。当快慢指针相遇时，再创建一个指针p指向链表头, 让p和slow等速前进，p和slow必定会相遇，且相遇点即为入环点。

数学证明参考：[LeetCode官方题解](https://leetcode-cn.com/problems/linked-list-cycle-ii/solution/huan-xing-lian-biao-ii-by-leetcode-solution/)

**复杂度分析：** 时间复杂度O(N), 空间复杂度O(1)

```C++
ListNode *detectCycle(ListNode *head) {
    ListNode *fast, *slow;
    fast = slow = head;
    while(fast) {
        if (fast->next == NULL) {
            return NULL;
        }
        // fast前进速度为slow两倍
        fast = fast->next->next;
        slow = slow->next;
        // 快慢指针相遇时，用一个指针p指向链表头，让p和slow同时前进，p和slow相遇点即为入环点，
        if (fast == slow) {
            ListNode *p = head;
            while (p != slow) {
                p = p->next;
                slow = slow->next;
            }
            return p;
        }
    }
    return NULL;
}
```



#### 输出链表中倒数第k个节点

以下给出两种思路：

**方法一：** 遍历两次单链表，第一次遍历求出链表长度记为len, 第二次遍历找到第len - k个节点并输出即可。

**复杂度分析：** 时间复杂度 O(N), 空间复杂度 O(1)

```C++
ListNode* getKthFromEnd(ListNode* head, int k) {
	int len = 0;
	ListNode *cur = head;
	while(cur != NULL) {
		++len;
		cur = cur->next;
	}
	for(int i = 0; i < len - k; ++i) {
		head = head->next;
	}
	return head;
}
```

**方法二：** 双指针法，设指针p, q，初始时将p指向链表头节点，q指向第k+1个节点，然后让p, q指针等速前进，当q为NULL时， p即为所求。

**复杂度分析：** 时间复杂度 O(N), 空间复杂度 O(1)

```C++
ListNode* getKthFromEnd(ListNode* head, int k) {
    ListNode *front = head;
    ListNode *behind = head;
    for (int i = 0; i < k; ++i) {
    	front = front->next;
    }
    while(front != NULL) {
        front = front->next;
        behind = behind->next;
    }
    return behind;
}
```



#### 反转链表

以下给出两种思路：

**方法一：** 遍历原链表，依次对每个节点用头插法创建新链表并返回。

**复杂度分析：** 时间复杂度 O(N), 空间复杂度O(N)

```C++
ListNode* reverseList(ListNode* head) {
    ListNode *l = NULL;
    while(head) {
        // 依次对每个节点用头插法创建新链表
        ListNode *s = new ListNode(head->val, NULL);
        if (l == NULL) {
            l = s;
        } else {
            s->next = l;
            l = s;
        }
        head = head->next;
    }
    return l;
}
```

**方法二：** 迭代法，遍历单链表，依次改变每个节点的next指向即可。注意next指向改变后，就无法访问下一个节点了，所以要在改next指针之前，用一个临时指针保存next指向的节点。

**复杂度分析：** 时间复杂度 O(N), 空间复杂度O(1)

```C++
ListNode* reverseList(ListNode* head) {
    ListNode *pre = NULL;
    ListNode *cur = head;
    for( ; cur != NULL ; ) {
        ListNode *tmpNext = cur->next; // 用临时指针暂存next指向的节点
        cur->next = pre;
        pre = cur;
        cur = tmpNext;	// 当前节点next值改变后，从临时指针恢复下一个需遍历的节点
    }
    return pre;
}
```
