---
title: LeetCode  Flatten a Multilevel Doubly Linked List
date: 2019-05-12 09:51:01
categories:
    - Algorithm
tags:
    - Algorithm
---
Flatten the list so that all the nodes appear in a single-level, doubly linked list. You are given the head of the first level of the list.

<!--more-->

题目： [430. Flatten a Multilevel Doubly Linked List]([Loading...](https://leetcode.com/problems/flatten-a-multilevel-doubly-linked-list/)
)

分析： 
![da80f35a.png](/img/ed6ee62a-3dcb-4189-9404-14b91692d436/da80f35a.png)

 当遇到有child的Node，先用`*next`存cur->next; 把cur->next指针指向child, child->pre指向cur； 接着以child为起点找到该曾最后一个Node，让它指向`*next`的节点。 接着以原来的child为cur Node重新开始一轮。 

```c++
    class Node {
    public:
        int val;
        Node *prev;
        Node *next;
        Node *child;

        Node() {}

        Node(int _val, Node *_prev, Node *_next, Node *_child) {
            val = _val;
            prev = _prev;
            next = _next;
            child = _child;
        }
    };

    Node *flatten(Node *head) {
        Node *cur = head;
        while (cur) {
            if (cur->child) {
                Node *next = cur->next;
                cur->next = cur->child;
                cur->child = nullptr;
                cur->next->prev = cur;

                // iter the children 
                Node *p = cur->next;
                while (p->next) p = p->next;
                  
                p->next = next;
                if (next) next->prev = p;
            }
            cur = cur->next;
        }

        return head;
    }
```

时间复杂度： $O(n)$