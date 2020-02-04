---
title: 'LeetCode:237. Delete Node in a Linked List'
date: 2019-05-12 12:52:10
categories:
    - Algorithm
tags:
    - Algorithm
---

Write a function to delete a node (except the tail) in a singly linked list, given only access to that node.

<!--more-->

## [题目](https://leetcode.com/problems/delete-node-in-a-linked-list/)

Write a function to delete a node (except the tail) in a singly linked list, given only access to that node.

Given linked list -- head = [4,5,1,9], which looks like following:

![237_example.png](https://assets.leetcode.com/uploads/2018/12/28/237_example.png)

## 分析： 

如果我们有一个如下的Linked List，想删除值是5的节点. 这道题要求 ，`only access to that node`。

```
4 -> 5 -> 1
```

![48d29b19.png](/img/f394f304-a25b-4128-bcaa-779a974c8c4a/48d29b19.png)
留意图中，要删除的对象的地址是0x7ff07ae00090

```c++
    void deleteNode(ListNode* node) {
        *node = *(node->next);
    }
```

`*node`对node指针取内容，返回是一个对象, 对象地址0x7ff07ae00090

![f94bba4f.png](/img/f394f304-a25b-4128-bcaa-779a974c8c4a/f94bba4f.png)

`*node->next`对node指针取内容，返回是一个对象， 对象地址0x7ff07ae000a0

![69411ad8.png](/img/f394f304-a25b-4128-bcaa-779a974c8c4a/69411ad8.png)

执行
```
*node = *(node->next);
```
结果如下： 

![83d8670d.png](/img/f394f304-a25b-4128-bcaa-779a974c8c4a/83d8670d.png)

node指针所执行的对象地址不变，但是对象内容发生变化。就像三个房子，位置没变， 但是第三个房子里的住户搬到了第二个房子里住了。 node指针指向的对象的地址依旧是0x7ff07ae00090, val不再是5而是1， next指针也执向NULL。但是我们也丢失了对0x7ff07ae000a0对象的指针引用。

![e8798a20.png](/img/f394f304-a25b-4128-bcaa-779a974c8c4a/e8798a20.png)