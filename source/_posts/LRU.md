---
title: LRU 缓存
date: 2023-09-26 22:04:10
tags:
  - algorithm
  - data structure
  - cache
  - lru
categories:
  - algorithm
  - data structure
  - cache
  - lru
---

## 背景
内存空间是有限的，当数据量超过空间的时候就得考虑淘汰哪部分数据，那按什么规则淘汰呢？

把最久没用到的数据淘汰了不就行了，最久没用的数据直觉上不就是不太会被访问的数据吗，既然不会被访问，那就先淘汰呗，LRU 算法就是这么来的

LRU（Least Recently Used）按字面意思理解就是**最近最少使用**，那该怎么知道哪个数据是最近最少使用的呢？

## LRU 的数据结构
### 用链表实现！
用链表是个很自然就可以想到的办法。每次数据块在存取完后将其重新排到头节点上，这样当数据在头节点说明它最近才被使用过

数据块在尾节点说明它是最久没被用到的，当数据超出内存限制时将尾节点的数据块删掉就行了

### 但链表有个缺点
就是链表查询、更新、删除操作的时间复杂度都是 O(n)，只有在头尾新增数据时才是 O(1)，要是能把 O(n) 复杂度都降到 O(1) 岂不妙哉？

### 用哈希表呀！
哈希表查询数据的复杂度不就是 O(1) 嘛！

那我们该怎么把哈希表用进去呢？

首先数据在查询时肯定需要从哈希表查，这样才能发挥出哈希表查得快的作用

其次要把数据被使用的信息体现到链表里（也就是把刚用过的数据提到头节点），这样才可以在数据溢出时知道去淘汰哪个

### 话不多说，3，2，1，上代码！
``` Go
package problems

type LRUCache struct {
	LinkedList    *DoubleLinkedList
	LinkedNodeMap map[int]*DoubleLinkedNode
	capacity      int
}

func Constructor(capacity int) LRUCache {
	if capacity == 0 {
		panic("capacity should not be zero")
	}
	return LRUCache{
		LinkedList:    ConstructDoubleLinkedList(),
		LinkedNodeMap: map[int]*DoubleLinkedNode{},
		capacity:      capacity,
	}
}

func (cache *LRUCache) Get(key int) int {
	node, ok := cache.LinkedNodeMap[key]
	if !ok {
		return -1
	}
	cache.LinkedList.MoveToTop(node)
	return node.Value.(CacheNode).Value
}

func (cache *LRUCache) Put(key int, value int) {
	newCacheNode := CacheNode{key, value}
	node, ok := cache.LinkedNodeMap[key]

	if ok {
		node.Value = newCacheNode
		cache.LinkedList.MoveToTop(node)
		return
	}

	if len(cache.LinkedNodeMap) == cache.capacity {
		delete(cache.LinkedNodeMap, cache.LinkedList.Tail.Pre.Value.(CacheNode).Key)
		cache.LinkedList.Remove(cache.LinkedList.Tail.Pre)
	}

	newLinkedNode := &DoubleLinkedNode{Value: newCacheNode}
	cache.LinkedNodeMap[key] = newLinkedNode
	cache.LinkedList.AddToTop(newLinkedNode)
}

type DoubleLinkedList struct {
	Head *DoubleLinkedNode
	Tail *DoubleLinkedNode
}

func ConstructDoubleLinkedList() *DoubleLinkedList {
	tail := &DoubleLinkedNode{}
	head := &DoubleLinkedNode{}
	tail.Pre = head
	head.Next = tail
	return &DoubleLinkedList{
		Head: head,
		Tail: tail,
	}
}

func (list *DoubleLinkedList) MoveToTop(node *DoubleLinkedNode) {
	list.Remove(node)
	list.AddToTop(node)
}

func (list *DoubleLinkedList) Remove(node *DoubleLinkedNode) {
	node.Pre.Next = node.Next
	node.Next.Pre = node.Pre
}

func (list *DoubleLinkedList) AddToTop(node *DoubleLinkedNode) {
	list.Head.Next.Pre = node
	node.Next = list.Head.Next
	node.Pre = list.Head
	list.Head.Next = node
}

type CacheNode struct {
	Key   int
	Value int
}
```
I have a 链表～, I have a 哈希表～, 嗯！？LRU Cache ！

怎么样，是不是很好玩呀
