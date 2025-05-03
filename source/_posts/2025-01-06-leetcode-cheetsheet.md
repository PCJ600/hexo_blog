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

## 是否包含子串
```
s2是否包含s1
s2.find(s1) != string::npos;
```

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

# 列表

* 初始化
```
v = [1,2,3] 
v = [0] * 5 # [0,0,0,0,0]
```
* 二维列表初始化
5行4列, 初始值为0
```
v = [[0 for _ in range(4)] for _ in range(5)]
```

* 末尾追加一个元素
```
>>> v.append(4)
>>> v
[1, 2, 3, 4]
```

* 末尾删除一个元素
```
>>> v.pop()
>>> v
[1,2,3]
```

* 指定位置插入元素(insert)
```
>>> v.insert(0, -1) 
[-1, 1, 2, 3, 4]
```

* 删除指定位置的元素(del)
```
>>> del v[0]
[1, 2, 3, 4]
```

* 删除指定位置的元素, 并获取被移除元素(pop)
```
>>> v
[1,3,5,7]
>>> e = v.pop(1)
>>> e
3
>>> v
[1,5,7]
```

* 删除第一个匹配的元素(remove)
```
>>> v.remove(2)
[1, 3, 4]
```

* 删除所有匹配元素
```
>>> v = [1,2,2,3,4,5]
>>> v = [x for x in v if x != 2]
>>> print(v)
[1,3,4,5]
```

* 正向遍历列表
```
for item in v:
    print(item)
```

* 倒序遍历列表
```
for item in reversed(v):
    print(item)
```

* 按下标遍历列表
```
for index in range(len(v)):
    print(v[index])
```

* 按下标倒序遍历列表
```
for idx in range(len(v)-1, -1, -1):
	print(v[idx])
```

* 正向遍历索引和值
```
for idx, val in enumerate(list):
	print(idx, val)
```

注: Python range函数
```
range(start, stop[, step])
* 从start开始, 到stop结束(不包括stop), step为步长, 默认值1)
```

* 一个列表追加到另一个列表
```
a = [1,2,3]
b = [4,5,6]
a.extend(b)
print(a)
[1,2,3,4,5,6]
```

* 列表排序
```
v.sort()
```

* 倒序排序
```
v.sort(reverse=True)
```

* 列表排序并去重
```
v = [3,3,2,2,1,1]
sorted_v = sorted(set(v))
print(sorted_v)
[1,2,3]
```

* 自定义排序

lambda表达式
```
points = [[1,3],[-2,2],[5,8],[0,1]]
sorted_points = sorted(points, key=lambda p: p[0]**2 + p[1]**2)
print(sorted_points)
```

普通函数
```
points = [[1,3],[-2,2],[5,8],[0,1]]
def get_distance_squared(point):
    return point[0] ** 2 + point[1] ** 2
sorted_points = sorted(points, key=get_distance_squared)
print(get_distance_squared)
```


# 链表

* 翻转链表
```
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
		
head = ListNode(1)
head.next = ListNode(2)
head.next.next = ListNode(3)

def reverse_list(head):
	prev = None
	curr = head
	while curr:
		next_temp = curr.next
		curr.next = prev
		prev = curr
		curr = next_temp
	return prev

new_head = reverse_list(head)
```

# 队列(deque)
```
from collections import deque
dq = deque()     # 创建队列
dq.append(x)     # 添加到右端
dq.appendleft(x) # 添加到左端
x = dq.pop()     # 删除右端
x = dq.popleft() # 删除左端
not dq           # 判断是否为空
len(dq)          # 获取长度
dq[0]            # 查看左端
dq[-1]           # 查看右端
dq.clear()       # 清空 
```

# 线程安全队列(queue.Queue)
```
from queue import Queue
q = Queue()
q.put(1)
q.put(2)
while not q.empty():
	item = q.get()
```

# 优先级队列(heapq)

默认最小堆
```
import heapq
heap = []
heap.heappush(heap, 3)
heap.heappush(heap, 1)
heap.heappush(heap, 2)
print(heap.heappop(heap))
```
注: 如果要模拟最大堆可以把数值取反后压入堆中

* 求第k大的数
```
def findKthLargest(nums: List[int], k: int) -> int:
	heap = []
	for x in nums:
		heapq.heappush(heap, -x)
	for i in range(k-1):
		heapq.heappop(heap)
	return -heapq.heappop(heap)
```

# 集合
```
s = {1,2,3}
s.add(x) # 添加元素
s.remove(x) # 删除元素x, 若不存在则报错
s.discard(x) # 删除元素x, 若不存在则忽略
s.clear() # 清空集合
if x in s: # 判断x是否在集合中
```

# 字典
```
dic = {}
person = {'name': Alice, 'age': 18}
person['name']
```
* 访问值
```
person.get('name', 'Unknown') # 获取key对应值, 如不存在返回None
person['name'] # 访问值
```
* 删除键值对
```
del person['name']
```
* 遍历字典
遍历key
```
for key in person:
	print(key)
for key in person.keys():
	print(key)
for val in person.values():
	print(val)
```

# 字符串
```
upper() 转大写
lower() 转小写
```
查找
```
text = 'hello world'
text.find('world') # 输出6, 未找到返回-1
text.replace('world', 'Python')
```
分割和连接
```
text = "apple,banana,cherry"
print(text.split(","))  # 输出: ['apple', 'banana', 'cherry']

words = ["hello", "world"]
print("-".join(words))  # 输出: hello-world
```
去除空白字符
```
text = "   hello world   "
print(text.strip())  # 输出: hello world
print(text.lstrip()) # 输出: hello world   
print(text.rstrip()) # 输出:    hello world
```
format格式化字符串
```
str = 'My name is {} and I am {} years old.'.format(name, age)
str2 = f'My name is {name} and I am {age} years old'
```
是否包含子串
```
text = 'hello world'
print('world' in text)
```
整数转字符串
```
str(123)
```