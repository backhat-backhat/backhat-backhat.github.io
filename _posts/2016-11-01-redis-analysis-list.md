---
layout: post
title: "redis源码分析 - 链表结构"
date: 2016-11-01 23:30:00
tags: redis list 数据结构
---
redis 链表解析

redis 中链表结构如下
{% highlight ruby %}
	typedef struct listNode {
	    struct listNode *prev;	//指向前驱指针
	    struct listNode *next;	//指向后继指针
	    void *value;
	} listNode;	//节点
	
	typedef struct list {
	    listNode *head;		//头结点
	    listNode *tail;		//尾节点
	    void *(*dup)(void *ptr);
	    void (*free)(void *ptr);
	    int (*match)(void *ptr, void *key);
	    unsigned long len;
	} list;		//链表
{% endhighlight %}

redis 的链表为双向链表，每一个链表结点同时具有一个指向前驱和指向其后继节点的指针，多个节点就组成了一个双向链表。

list 结构为链表提供了头指针 head 和尾指针 tail，以及链表长度计数器 len，free 和 match 则用于实现多态链表所需的类型特定函数: <br>

* dup 用于复制节点所保存的值
* free 用于释放节点所保存的值
* match 用于比对节点所保存的值与另一个输入值是否相等

## 创建链表
{% highlight ruby %}
	/* Create a new list. The created list can be freed with
	 * AlFreeList(), but private value of every node need to be freed
	 * by the user before to call AlFreeList().
	 *
	 * On error, NULL is returned. Otherwise the pointer to the new list. */
	list *listCreate(void)
	{
	    struct list *list;
	
	    if ((list = zmalloc(sizeof(*list))) == NULL)
	        return NULL;
	    list->head = list->tail = NULL;
	    list->len = 0;
	    list->dup = NULL;
	    list->free = NULL;
	    list->match = NULL;
	    return list;
	}
{% endhighlight %}

创建链表，链表没有节点，头尾指针均指向`NULL`

## 从链表头或者链表尾插入新节点
{% highlight ruby %}
	/* Add a new node to the list, to head, containing the specified 'value'
	 * pointer as value.
	 *
	 * On error, NULL is returned and no operation is performed (i.e. the
	 * list remains unaltered).
	 * On success the 'list' pointer you pass to the function is returned. */
	list *listAddNodeHead(list *list, void *value)
	{
	    listNode *node;
	
	    if ((node = zmalloc(sizeof(*node))) == NULL)
	        return NULL;
	    node->value = value;
	    if (list->len == 0) {	//链表为空
	        list->head = list->tail = node;
	        node->prev = node->next = NULL;
	    } else {	//从头插入，头节点的 prev 指向 NULL
	        node->prev = NULL;
	        node->next = list->head;
	        list->head->prev = node;
	        list->head = node;
	    }
	    list->len++;
	    return list;
	}
	
	/* Add a new node to the list, to tail, containing the specified 'value'
	 * pointer as value.
	 *
	 * On error, NULL is returned and no operation is performed (i.e. the
	 * list remains unaltered).
	 * On success the 'list' pointer you pass to the function is returned. */
	list *listAddNodeTail(list *list, void *value)
	{
	    listNode *node;
	
	    if ((node = zmalloc(sizeof(*node))) == NULL)
	        return NULL;
	    node->value = value;
	    if (list->len == 0) {
	        list->head = list->tail = node;
	        node->prev = node->next = NULL;
	    } else {	//从尾插入，尾指针的 next 指向 NULL
	        node->prev = list->tail;
	        node->next = NULL;
	        list->tail->next = node;
	        list->tail = node;
	    }
	    list->len++;
	    return list;
	}
{% endhighlight %}
## 在指定节点处添加新节点
{% highlight ruby %}
	list *listInsertNode(list *list, listNode *old_node, void *value, int after) {
	    listNode *node;
	
	    if ((node = zmalloc(sizeof(*node))) == NULL)
	        return NULL;
	    node->value = value;
	    if (after) {	//在 old_node 节点后插入节点
	        node->prev = old_node;
	        node->next = old_node->next;
	        if (list->tail == old_node) {
	            list->tail = node;
	        }
	    } else {	//在old_node节点前插入节点
	        node->next = old_node;
	        node->prev = old_node->prev;
	        if (list->head == old_node) {
	            list->head = node;
	        }
	    }
	    if (node->prev != NULL) {
	        node->prev->next = node;
	    }
	    if (node->next != NULL) {
	        node->next->prev = node;
	    }
	    list->len++;
	    return list;
	}
{% endhighlight %}
## 旋转链表，将链表尾节点插入到头部
{% highlight ruby %}
	/* Rotate the list removing the tail node and inserting it to the head. */
	void listRotate(list *list) {
	    listNode *tail = list->tail;
	
	    if (listLength(list) <= 1) return;
	
	    /* Detach current tail */
		// 将尾节点分离出来
	    list->tail = tail->prev;
	    list->tail->next = NULL;
	    /* Move it as head */
		// 在头部插入尾节点
	    list->head->prev = tail;
	    tail->prev = NULL;
	    tail->next = list->head;
	    list->head = tail;
	}
{% endhighlight %}
## 释放链表
{% highlight ruby %}
	/* Free the whole list.
	 *
	 * This function can't fail. */
	void listRelease(list *list)
	{
	    unsigned long len;
	    listNode *current, *next;
	
	    current = list->head;
	    len = list->len;
	    while(len--) {
	        next = current->next;
	        if (list->free) list->free(current->value);
	        zfree(current);
	        current = next;
	    }
	    zfree(list);
	}
{% endhighlight %}
	
redis 双向链表的特性： <br>

* 链表节点带有 prev 和 next 指针，获取某节点前驱或者后继节点的时间复杂度为 O(1)
* 无环：链表头节点的 prev 指针和尾节点的 next 指针均指向 NULL，即从头遍历或者从尾遍历都是以 NULL 结束链表遍历，不会出现环。
* 头尾指针使得获取链表头尾指针的时间复杂度为 O(1)
* list 的 长度计数器 len，使得获取链表长度的时间复杂度为 O(1)
* 链表结点通过 `void*` 来保存节点值，可以接受任何类型的数据，同时，free，match 和 dup 三个属性为节点值设置类型特定函数，使得链表可以用于保存各种不同类型的值。