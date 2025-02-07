---
layout: next
title: LeetCode cheetsheet for C++
date: 2025-01-06 20:19:17
categories: LeetCode
tags: LeetCode
---


# 数组(vector)

## 初始化
```
vector<int> v(size);
vector<int> v(size, init_value);
v.emplace_back(a.begin(), a.begin() + 3); # 用另一个vector的某一区间初始化vector
```


## 遍历数组
```
for (int i = 0; i < vec.size(); ++i) {
	vec[i] = i;
}
```

## 数组追加到另一个数组
```
vec1.insert(vec1.end(), vec2.begin(), vec2.end());
```

## 排序数组+去重
```
sort(nums.begin(), nums.end());
nums.erase(unique(nums.begin(), nums.end()), nums.end());
```

## 逆序排序
```
sort(nums.begin(), nums.end(), greater<int>());
```

## 二维vector初始化
```
vector<vector<bool>> visited(rows, vector<bool>(cols, false));
```
<!-- more -->

# 链表

## 反转链表
```
    ListNode* reverseList(ListNode* head) {
        if (head == nullptr || head->next == nullptr) return head;

        ListNode *prev = nullptr;
        ListNode *cur = head;
        ListNode *tmp = nullptr;
        while (cur != nullptr) {
            tmp = cur->next;
            cur->next = prev;
            prev = cur;
            cur = tmp;
        }
        return prev;
    }
```

## 快慢指针找中间节点
```c++
class Solution {
public:
    ListNode* middleNode(ListNode* head) {
        if (head == nullptr || head->next = nullptr) return head;
        ListNode *fast = head;
        ListNode *slow = head;
        while (fast != nullptr) {
            fast = fast->next;
            if (fast != nullptr) {
                fast = fast->next;
				slow = slow->next;
            }
        }
        return slow;
    }
};
```

## 双向链表节点定义
```c++
struct DLinkedNode {
    DLinkedNode *prev;
    DLinkedNode *next;
    int key;
    int value;
    DLinkedNode(): key(0), value(0), prev(nullptr), next(nullptr) {}
    DLinkedNode(int k, int v): key(k), value(v), prev(nullptr), next(nullptr) {}
};
```

# 队列(Queue)
```
queue<int> q;
q.front() #队首
q.push()
q.pop()
q.size()
q.empty()
```

# 双端队列(Deque)
```
deque<int> dq;
dq.push_back(x);
dq.pop_back();
dq.pop_front();
dq.front();
dq.back();
for (auto iter = dq.begin(); iter != dq.end(); ++iter) {
	*iter
}
```

# 优先级队列/堆(priority_queue)

## 求第k大的数
* 大根堆: 建大小为n的堆, 弹出k-1次，堆顶为第k大的数
* 小根堆: 维护大小为k的堆，取堆顶元素

大根堆(默认)
```c++
int findKthLargest(vector<int> &nums, int k) {
	priority_queue<int, vector<int>, less<int>> pq;
	for (int i = 0; i < nums.size(); ++i) {
		pq.push(nums[i]);
	}
	for (int i = 0; i < k - 1; ++i) {
		pq.pop();
	}
	return pq.top();
}
```

小根堆
```c++
    int findKthLargest(vector<int> &nums, int k) {
        priority_queue<int, vector<int>, greater<int>> pq;
        for (int i = 0; i < k; ++i) {
            pq.push(nums[i]);
        }
        for (int i = k; i < nums.size(); ++i) {
            pq.push(nums[i]);
            pq.pop();
        }
        return pq.top();
    }
```

## 自定义优先级
求k个最小距离的点
```
class Solution {
public:
    struct cmp{
        bool operator() (vector<int> &a, vector<int> &b) {
            int dis1 = a[0] * a[0] + a[1] * a[1];
            int dis2 = b[0] * b[0] + b[1] * b[1];
            return dis1 > dis2;
        }
    };
    vector<vector<int>> kClosest(vector<vector<int>>& points, int k) {
        vector<vector<int>> ans;
        priority_queue<vector<int>, vector<vector<int>>, cmp> pq;
        for (auto &vec: points) {
            pq.push(vec);
        }
        for (int i = 0; i < k; ++i) {
            ans.push_back(pq.top());
            pq.pop();
        }
        return ans;
    }
};
```

# 集合(unordered_set)
```
unordered_set<int> s;
s.insert(key);
s.erase(key);
if (s.find(key) != s.end())
```

# 哈希表(unordered_map)
```
unordered_map<int, DLinkedNode*> ht;
ht.erase(key);
```

# TreeMap
## 自定义优先级
```
class Cmp {
    private:
        const std::vector<int>& data;
    public:
        explicit Cmp(const std::vector<int> &vec) : data(vec) {}
        bool operator()(const int a, const int b) const {
            if (data[a] != data[b]) return data[a] > data[b];
            return a > b;
        }        
};
vector<int>& nums;
set<int, Cmp> s((Cmp(nums)));
```

# 字符串(String)

## 比较两个字符串的字典序
```
s1 < s2
s1.compare(s2) < 0
```

## 整数转字符串
```
string s = to_string(10);
```

## 去除所有空格
```

```

## 末尾添加字符
```
string s;
s.push_back('c');
```

# 排序

## 自定义排序
```c++
static bool compare(vector<int> &a, vector<int> &b) {
    return a[0] < b[0];
}
sort(intervals.begin(), intervals.end(), compare);
```

## 拓扑排序
BFS, 每次把入度为0的点入队列)
```
vector<int> degree(size, 0);
vector<vector<int>> graph(size);
queue<int> q;
```

# 并查集
```
vector<int> parents;
for (int i = 0; i < parents.size(); ++i) parents[i] = i;
void Union(int x, int y) {
	int px = Find(x);
	int py = Find(y);
	if (px == py) return;
	parents[px] = py;
}
int Find(int x) {
	if (x == parents[x]) return x;
	parents[x] = Find(parents[x]);
	return parents[x];
}
```

# 字典树
```
struct TrieNode {
    vector<TrieNode *> children;
	bool isWordEnd;
    TrieNode() {
        children.resize(26, nullptr);
    }
};

void insert(string word) {
    TrieNode *cur = root;
    for (char ch: word) {
        int idx = ch - 'a';
        if (cur->children[idx] == nullptr) {
            cur->children[idx] = new TrieNode(ch);
        }
        cur = cur->children[idx];
    }
}

bool startsWith(string prefix) {
    TrieNode *cur = root;
    for (char ch: prefix) {
        int idx = ch - 'a';
        if (cur->children[idx] == nullptr) {
            return false;
        }
        cur = cur->children[idx];
    }
    return true;
}
```

# 单调栈、树状数组


# LeetCode python CheetSheet

# 自定义排序
Python匿名函数
```py
# intervals: List[List[int]])
intervals.sort(key=lambda p: p[0])
intervals.sort(key=lambda p: p[0], reverse=True)
```
Python命名函数排序
```py
def compare(x, y):
	s1 = str(x) + str(y)
	s2 = str(y) + str(x)
	if s1 > s2:
		return -1
	return 1
nums.sort(key=cmp_to_key(compare))
```

# 遍历list
enumerate正向遍历索引和值
```py
for index,value in enumerate(list):
```
range逆序遍历 (起始索引(含)，结束索引(不含), 步长)
```py
for idx in range(len(nums)-1, -1, -1):
	nums[idx] = ...
```

# 小根堆
Python heapq默认是小根堆
```
class Solution:
    def findKthLargest(self, nums: List[int], k: int) -> int:
        min_heap = []
        for i in range(k):
            heapq.heappush(min_heap, nums[i])
        for i in range(k, len(nums), 1):
            heapq.heappush(min_heap, nums[i])
            heapq.heappop(min_heap)
        return min_heap[0]
```

# 集合(set)
```
s = set()
s.add(e)
if e in s:
	...
```

# 队列
```
from collections import deque
d = deque()
d = deque([1, 2, 3])

d.append(4)
d.appendleft(0)
removed_element = d.pop()
removed_element = d.popleft()
d[0]
d[-1]
d.remove(2)
```