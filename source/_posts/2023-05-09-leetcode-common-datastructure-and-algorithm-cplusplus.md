---
layout: next
title: LeetCode 常用数据结构和算法(C++)
date: 2023-05-09 21:41:16
categories: LeetCode
tags: LeetCode
---

## list
```c++
list<int> list;
void list::splice( iterator pos, list rhs );				// 全部移动
void list::splice( iterator pos, list rhs, iterator ix );	// 移一个
void list::splice( iterator pos, list rhs, iterator first, iterator last );	// 移一组
```

<!-- more -->
## pair
```c++
pair<int, int> v = {1, 2}
v.first
v.second
```
## stack
```c++
stack<int> v;
```
## deque
```c++
deque<pair<int, int>> q;
q.front()
q.back()
q.pop_front()
q.pop_back()
q.push_back()
q.push_front()
```
```python
from collections import deque
queue = deque(["Eric", "John", "Michael"])
queue.append("Graham")
queue.popleft()
```
## queue
```c++
queue<int> q;
q.push(x);
q.front();
q.pop();
q.top();
```
```python
q=Queue(maxsize=5)  # 先进先出
q=LifoQueue()		# 后进先出
pq=PriorityQueue()	# 优先级队列
p.put(i)
p.empty()
q.qsize()
q.full()
q.get()
```
## heap
```c++ 大根堆
    int findKthLargest(vector<int>& nums, int k) {
        priority_queue<int, vector<int>, less<int>> h;
        for (int i = 0; i < nums.size(); ++i) {
            h.push(nums[i]);
        }

        for (int i = 0; i < k - 1; ++i) {
            h.pop();
        }
        return h.top();
    }
```
```python
# https://blog.csdn.net/weixin_43790276/article/details/107741332
import heapq
    def findKthLargest(self, nums: List[int], k: int) -> int:
        heap = []
        n = len(nums)
        for num in nums:
            heapq.heappush(heap, num)
        for i in range(n - k):
            heapq.heappop(heap)
        return heap[0]
```
## hash
```c++
unordered_map<int, int> ht;
ht.find(key) != ht.end() // exist
iter->first()
iter->second()
```
```python
dic = {}
if key in dic:
rows = set()
rows.add(1)
rows.remove(2)
rows.clear()
```
## sort
```c++
#include <algorithm>
sort(arr.begin(), arr.end());
sort(myvector.begin(), myvector.begin() + 4, std::greater<int>());
```
```python
# intervals: List[List[int]]
def cmp(listx, listy):
    if listx[0] == listy[0]:
        return listx[1] - listy[1]
    return listx[0] - listy[0]
intervals.sort(key=cmp_to_key(cmp))
```
## string sort
```py
sorted_str = ''.join(sorted(strs[i]))
```
## custom sort
```c++
static bool Cmp(vector<int> &v1, vector<int> &v2) {
    if (v1[0] == v2[0]) return v1[1] < v2[1];
    return v1[0] < v2[0];
}
sort(intervals.begin(), intervals.end(), Cmp);
```
```python
from functools import cmp_to_key
lst = [(9, 4), (2, 10), (4, 3), (3, 6)]

def cmp(x, y):
    a = x[0] if x[0] %2 == 1 else x[1]
    b = y[0] if y[0] %2 == 1 else y[1]
    return 1 if a > b else -1 if a < b else 0
lst.sort(key=cmp_to_key(cmp))
print(lst)
```
