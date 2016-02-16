---
layout: post
title: Lists of Linux kernel
date: 2016-02-16
categories: [blog]
tags: [Linux kernel,list,hlist, ]
description: Linux 内核中使用的链表的结构，hash 表的结构，以及相关的操作函数说明
---

{{ more }}

### 目录
{:.no_toc}
* Table of Contents Placeholder
{:toc}
----------


最近想看一下Linux kernel的IP协议栈相关源代码，发现首先得做很多功课，比如数据结构。初步看了下list的实现，的确有很多奇妙的地方。源代码位于`include/linux/list.h`，新版本内核可能会添加少许函数，基本的函数都是一致的。

## 0x00 如何使用list（hlist）

平常使用list的时候都是在链表中包含数据，比如

>```
>
>struct test_list {
>	struct test_list *next;
>	int data
>};
>
>```

内核中使用list时，是在数据中包含链表，比如

```
struct nf_sockopt_ops
{
	struct list_head list;

	int pf;

	/* Non-inclusive ranges: use 0/0/NULL to never get called. */
	int set_optmin;
	int set_optmax;
	int (*set)(struct sock *sk, int optval, void __user *user, unsigned int len);
	int (*compat_set)(struct sock *sk, int optval,
			void __user *user, unsigned int len);

	int get_optmin;
	int get_optmax;
	int (*get)(struct sock *sk, int optval, void __user *user, int *len);
	int (*compat_get)(struct sock *sk, int optval,
			void __user *user, int *len);

	/* Number of users inside set() or get(). */
	unsigned int use;
	struct task_struct *cleanup_task;
};

	//definition:
	static LIST_HEAD(nf_sockopts);

	//list add
	list_add(&reg->list, &nf_sockopts);




```

有哪些优点呢？


> * Type Oblivious:
> This list can be used to strung up any data structure you have in mind.
>
>* Portable:
>Though I haven't tried in every platform it is safe to assume the list implementation is very portable. Otherwise it would not have made it into the kernel source tree.
>
>* Easy to Use:
>Since the list is type oblivious same functions are used to initialize, access, and traverse any list of items strung together using this list implementation.
>
>* Readable:
>The macros and inlined functions of the list implementation makes the resulting code very elegant and readable.
>
>* Saves Time:
>Stops you from reinventing the wheel. Using the list really saves a lot of debugging time >and repetitively creating lists for every data structure you need to link.

甚至：

>1. List is inside the data item you want to link together.
>2. You **can put** struct list_head **anywhere** in your structure.
>3. You **can name** struct list_head **variable anything** you wish.
>4. You **can have** multiple lists!


因此，这样的方式非常灵活。

## 0x01 list

结构体定义：

>```
>
>struct list_head {
>	struct list_head *next, *prev;
>};
>
>```

这是一个双循环链表，提供初始化、添加、删除、移动、合并等链表的操作，里边的实现应该是比较自然直观的，除了函数末尾有rcu的函数之外，带rcu系列的**应该**是有预存取操作加速的。


## 0x02 hlist

结构体定义：

>```
>
>/*
> * Double linked lists with a single pointer list head.
> * Mostly useful for hash tables where the two pointer list head is
> * too wasteful.
> * You lose the ability to access the tail in O(1).
> */
>struct hlist_head {
>	struct hlist_node *first;
>};
>
>struct hlist_node {
>	struct hlist_node *next, **pprev;
>};
>
>```

因为hash表会很大，head部分只有一个指针，节省空间（注解里说的比较明白），还有其他考虑吗？

对于结构体的定义，第一眼的感觉，很多人可能会问，我也不能免俗，为什么不是：

```
struct hlist_head {
	struct hlist_node *first;
};

struct hlist_node {
	struct hlist_node *next, *prev;
};
```

写这篇文章其实主要就是为了消除这个疑惑的。

>由于hlist不是一个完整的循环链表，在list中，表头和结点是同一个数据结构，直接用prev是ok的。在hlist中，表头中没有prev，只有一个first。
>
>1. 链表head只有一个指向第一个节点的指针，链表节点分别有指向下一个和上一个节点的指针，如此上一个节点便不能和下一个节点使用相同的类型，因为第一个节点的上一个节点是一个struct hlist_head类型而不是hlist_node类型，于是这里巧妙地使用了指向上一个节点的**next指针的地址**作为上一个节点的指针，我们知道在获取上一个节点的时候一般是为了对它进行插入操作，而插入操作只需要操作上一个节点的next指针（hlist_head和hlist_node的指向下个节点的指针类型相同，这样便可以在插入和删除操作对于首节点和普通节点不失通用性了），而不需要操作prev指针，于是这种设计便足够使用了，为上一个节点的next指针赋值时只需要为*(node->pprev)赋值即可。
>
>2. 还解决了数据结构不一致，hlist_node巧妙的将pprev指向上一个节点的next指针的地址，由于hlist_head和hlist_node指向的下一个节点的指针类型相同，就解决了通用性。
>

如果到这里你还没看明白是怎么回事，那么给几个比较的代码就能明白了。

```

Part1
-------

//Use Linux kernel's way:
static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h)
{
	struct hlist_node *first = h->first;
	n->next = first;
	if (first)
		first->pprev = &n->next;
	h->first = n;
	n->pprev = &h->first;
}


//Use your own way:
Void hlist_add_head(struct hlist_node *n ,struct hlist_head *h) 
{
  struct hlist_node *first = h->first;

  n->next = first;
  if (first) 
    first->prev = n;
  h->first = n;
  n->prev = NULL;
}


Part2
---------
//Use Linux kernel's way:
/*insert next before n */
/* next must be != NULL */
static inline void hlist_add_before(struct hlist_node *n,
					struct hlist_node *next)
{
	n->pprev = next->pprev;
	n->next = next;
	next->pprev = &n->next;
	*(n->pprev) = n;
}


//Use your own way:
static inline void hlist_add_before(struct hlist_node *n,
					struct hlist_node *next,
					struct hlist_head *head) //extra push
{
	n->prev = next->prev;
	n->next = next;
	next->prev = n;
	if(n->prev){
		n->prev->next = n;
	}else{
		Head->first = n;
	}
}

```

看到没有，对于add函数，就要多出一个参数的push还有if-else的判断，远没有内核中实现的那么流畅和简洁！对于插入和删除等操作也是如此，因此，内核中实现的hlist是何等的简洁和高明，在此对写出此代码的人表示最真诚的敬意！


## 0x02 preserve






References
========

1. [Linux kernel design patterns - part 2](http://lwn.net/Articles/336255/)
2. [Linux kernel design patterns - part 1](http://lwn.net/Articles/336224/)
3. [Linux Kernel Linked List Explained](http://isis.poly.edu/kulesh/stuff/src/klist/)
4. [《Linux内核设计与实现》读书笔记（六）- 内核数据结构](http://www.cnblogs.com/wang_yb/archive/2013/04/16/3023892.html)


